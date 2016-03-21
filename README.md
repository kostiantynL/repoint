# repoint

## What is this and why do I need it?
A small library for generating functions to make requests to the backend REST APIs for your client applications. Imagine you have an REST API written by backend team which has a lof of endpoints. You could go ahead and write ad-hoc ajax requests in "jquery" style:

```js
$.ajax("/users")
  .done(function() {
    // success
  })
  .fail(function() {
    // error
  })
```

It will work, but there are a couple of problems here:

1. *URL hardcoding.* What if you want to use the same request, but with different "success logic" in another place of your app? You would need to copy-paste it. And what if URL in backend API is changed, - you would need to go and replace the URL everywhere where you copy-pasted it.
2. *It is not composable.* You have pieces of logic scattered all across the codebase. It would be better to have them in one place in some modular way, so you can have a client library for your REST API.
3. *Boilerplate.* Even if you move them into separate modules/functions you would still have to write a lof of boilerplate code, which is exhausting and error-prone.

This library aims to solve above problems.

## Important note
Repoint uses [isomorphic-fetch](https://github.com/matthew-andrews/isomorphic-fetch) under the hood, which requires ES6 Promise compatible polyfill.
For more information refer to https://github.com/matthew-andrews/isomorphic-fetch#warnings

## Installation
```sh
 npm install --save repoint
```
```sh
 npm install --save es6-promise
```
or any other polyfill

## Basic Usage
```js
import Repoint from 'repoint'

const repoint = new Repoint({ host: 'http://api.example.com/v1' })
const usersAPI = repoint.generate('users')

// makes GET request to http://api.example.com/v1/users
usersAPI.getCollection({})

// makes GET request to http://api.example.com/v1/users?some=1
usersAPI.getCollection({ some: 1 })

// makes GET request to http://api.example.com/v1/users/1
usersAPI.get({ id: 1 })

// makes POST request to http://api.example.com/v1/users with body { email: 'email@example.com' } and ContentType: application/json
usersAPI.create({ email: 'email@example.com' })

// makes PUT request to http://api.example.com/v1/users/1 with body { email: 'updated-email@example.com' } and ContentType: application/json
usersAPI.update({ id: 1, user: { email: 'updated-email@example.com' }})

// makes DELETE request to http://api.example.com/v1/users/1
usersAPI.destroy({ id: 1 })
```

You can change idAttribute key for member URLs (get, update, destroy) by providing `idAttribute` option, the default value is 'id'

```js
import Repoint from 'repoint'

const repoint = new Repoint({ host: 'http://api.example.com/v1' })
const usersAPI = repoint.generate('users', { idAttribute: 'slug' })

// makes GET request to http://api.example.com/v1/users/tom
usersAPI.get({ slug: 'tom' })
```

You can provide namespace as well:

```js
import Repoint from 'repoint'

const repoint = new Repoint({ host: 'http://api.example.com/v1' })
const usersAPI = repoint.generate('users', { idAttribute: 'slug', namespace: 'admin' })

// makes GET request to http://api.example.com/v1/admin/users/tom
usersAPI.get({ slug: 'tom' })
```

## Nested endpoints

To make requests to the nested route you use nestUnder option, which expects another endpoint:
```js
import Repoint from 'repoint'

const repoint = new Repoint({ host: 'http://api.example.com/v1' })
const postsAPI = repoint.generate('posts', { nestUnder: repoint.generate('users') })

// makes GET request to http://api.example.com/v1/users/1/posts
// userId is generated by the following pattern:
// singularized endpoint name + capitalized idAttribute
// in this case, it is "user" + "Id"
postsAPI.getCollection({ userId: 1 })

// makes GET request to http://api.example.com/v1/users/1/posts/1
postsAPI.get({ userId: 1, id: 1 })
```

There is no limit for nesting level. `idAttribute` and `namespace` options also work for nested endpoints

## Non-restful endpoints

It's often that you need to make request to a non-restful endpoint, e.g. /users/login

You can define them in the 3rd argument of generate function:

```js
import Repoint from 'repoint'

const repoint = new Repoint({ host: 'http://api.example.com/v1' })
const usersAPI = repoint.generate('users', {}, [{ method: 'post', name: 'login', on: 'collection' }])

// makes POST request to http://api.example.com/v1/users/login
usersAPI.login({ /* params */ })
```

## Singular endpoints

Those are URLs with no collectionURL and with no idAttributes (similar to Rails `resource` method in routes.rb)

You can define them in the following way:

```js
import Repoint from 'repoint'

const repoint = new Repoint({ host: 'http://api.example.com/v1' })
const userAPI = repoint.generate('user', { singular: true })

// makes GET request to http://api.example.com/v1/user
userAPI.get({}) // no `id` to get the user
```

This will generate only `get`, `create`, `update` and `destroy` functions omitting `getCollection`. Also no idAttributes are necessary.

Notice that you are providing a 'singular' name, (of course you can provide any name, pluralization/singularization simply is not applied in this case).

## Response decorators

When creating a Repoint instance, there is an additional config to transform the response for all the generated endpoints. By default, it returns what comes from the server as it is, but you can provide your own. I like to use [humps](https://github.com/domchristie/humps) to convert all of the object keys to camelCase or vice versa. You can do whatever you want, the only requirement is that it should be a function.

```js
import Repoint from 'repoint'
import { camelizeKeys, decamelizeKeys } from 'humps'

const repoint = new Repoint({
  host: 'http://api.example.com/v1',
  // convert query params or body params keys (in case of POST/PATCH/PUT requests) to underscore-separated
  paramsTransform: (data) => decamelizeKeys(data),
  beforeSuccess: (data) => camelizeKeys(data),
  /* beforeError: same but for error */
})
const usersAPI = repoint.generate('users')
usersAPI.get({ id: 1, firstName: 'Bob' /* will be converted to { id: 1, first_name: 'Bob' } */ })
        .then((data) => /* data will have all of its object keys converted to camelCase */)
```
