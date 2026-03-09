---
title: experimental_taintObjectReference
version: experimental
---

<Experimental>

**この API は実験的なものであり、まだ安定版の React では利用できません。**

React のパッケージを最新の experimental 版にアップグレードすることで試すことができます。

- `react@experimental`
- `react-dom@experimental`
- `eslint-plugin-react-hooks@experimental`

React の experimental 版にはバグが含まれている可能性があります。本番環境では使用しないでください。

この API は React Server Components 内でのみ利用できます。

</Experimental>


<Intro>

`taintObjectReference` は `user` オブジェクトのような特定のオブジェクトインスタンスが Client Component に渡されるのを防ぐためのものです。

```js
experimental_taintObjectReference(message, object);
```

キー、ハッシュ、トークンが渡されるのを防ぎたい場合は、[`taintUniqueValue`](/reference/react/experimental_taintUniqueValue) を参照してください。

</Intro>

<InlineToc />

---

## リファレンス {/*reference*/}

### `taintObjectReference(message, object)` {/*taintobjectreference*/}

`taintObjectReference` をオブジェクトと共に呼び出すことで、そのオブジェクトを「そのまま Client に渡してはならないもの」として React に登録します。

```js
import {experimental_taintObjectReference} from 'react';

experimental_taintObjectReference(
  'Do not pass ALL environment variables to the client.',
  process.env
);
```

[さらに例を見る](#usage)

#### 引数 {/*parameters*/}

* `message`: オブジェクトが Client Component に渡されてしまった場合に表示したいメッセージです。このメッセージは、そのオブジェクトが Client Component に渡されたときに投げられる Error の一部として表示されます。

* `object`: The object to be tainted. Functions and class instances can be passed to `taintObjectReference` as `object`. Functions and classes are already blocked from being passed to Client Components but the React's default error message will be replaced by what you defined in `message`. When a specific instance of a Typed Array is passed to `taintObjectReference` as `object`, any other copies of the Typed Array will not be tainted.

#### Returns {/*returns*/}

`experimental_taintObjectReference` returns `undefined`.

#### Caveats {/*caveats*/}

- Recreating or cloning a tainted object creates a new untainted object which may contain sensitive data. For example, if you have a tainted `user` object, `const userInfo = {name: user.name, ssn: user.ssn}` or `{...user}` will create new objects which are not tainted. `taintObjectReference` only protects against simple mistakes when the object is passed through to a Client Component unchanged.

<Pitfall>

**Do not rely on just tainting for security.** Tainting an object doesn't prevent leaking of every possible derived value. For example, the clone of a tainted object will create a new untainted object. Using data from a tainted object (e.g. `{secret: taintedObj.secret}`) will create a new value or object that is not tainted. Tainting is a layer of protection; a secure app will have multiple layers of protection, well designed APIs, and isolation patterns.

</Pitfall>

---

## 使用法 {/*usage*/}

### Prevent user data from unintentionally reaching the client {/*prevent-user-data-from-unintentionally-reaching-the-client*/}

A Client Component should never accept objects that carry sensitive data. Ideally, the data fetching functions should not expose data that the current user should not have access to. Sometimes mistakes happen during refactoring. To protect against these mistakes happening down the line we can "taint" the user object in our data API.

```js
import {experimental_taintObjectReference} from 'react';

export async function getUser(id) {
  const user = await db`SELECT * FROM users WHERE id = ${id}`;
  experimental_taintObjectReference(
    'Do not pass the entire user object to the client. ' +
      'Instead, pick off the specific properties you need for this use case.',
    user,
  );
  return user;
}
```

Now whenever anyone tries to pass this object to a Client Component, an error will be thrown with the passed in error message instead.

<DeepDive>

#### Protecting against leaks in data fetching {/*protecting-against-leaks-in-data-fetching*/}

If you're running a Server Components environment that has access to sensitive data, you have to be careful not to pass objects straight through:

```js
// api.js
export async function getUser(id) {
  const user = await db`SELECT * FROM users WHERE id = ${id}`;
  return user;
}
```

```js
import { getUser } from 'api.js';
import { InfoCard } from 'components.js';

export async function Profile(props) {
  const user = await getUser(props.userId);
  // DO NOT DO THIS
  return <InfoCard user={user} />;
}
```

```js
// components.js
"use client";

export async function InfoCard({ user }) {
  return <div>{user.name}</div>;
}
```

Ideally, the `getUser` should not expose data that the current user should not have access to. To prevent passing the `user` object to a Client Component down the line we can "taint" the user object:


```js
// api.js
import {experimental_taintObjectReference} from 'react';

export async function getUser(id) {
  const user = await db`SELECT * FROM users WHERE id = ${id}`;
  experimental_taintObjectReference(
    'Do not pass the entire user object to the client. ' +
      'Instead, pick off the specific properties you need for this use case.',
    user,
  );
  return user;
}
```

Now if anyone tries to pass the `user` object to a Client Component, an error will be thrown with the passed in error message.

</DeepDive>
