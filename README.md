# google-function-authorizer

[![Build Status](https://semaphoreci.com/api/v1/pauloddr/google-function-authorizer/branches/master/shields_badge.svg)](https://semaphoreci.com/pauloddr/google-function-authorizer)
[![Test Coverage](https://lima.codeclimate.com/github/pauloddr/google-function-authorizer/badges/coverage.svg)](https://lima.codeclimate.com/github/pauloddr/google-function-authorizer/coverage)
[![Code Climate](https://lima.codeclimate.com/github/pauloddr/google-function-authorizer/badges/gpa.svg)](https://lima.codeclimate.com/github/pauloddr/google-function-authorizer)

__(Currently under development)__

A simple user authentication and management system for [Google Cloud HTTP Functions](https://cloud.google.com/functions/docs/writing/http).

It stores user data in [Google Datastore](https://cloud.google.com/datastore/) using [gstore-node](https://github.com/sebelga/gstore-node); hashes passwords with [bcrypt](https://github.com/kelektiv/node.bcrypt.js); and manages browser sessions with [client-session](https://github.com/mozilla/node-client-sessions).

## Usage

Create a new Google HTTP Function with the following code:

```javascript
const users = require('google-function-authorizer');

exports.handleRequest = function (req, res) {
  users.handle(req, res);
};
```

Make sure the library is added to __package.json__, and that the __entry point__ is correct (in the example above, it should be `handleRequest`).

Then, assuming you named your function "users", the following endpoints will be served by your function:

* `POST /users`
* `POST /users/signin`
* `POST /users/signout`
* `GET /users`
* `GET /users/:id`
* `PUT|PATCH /users/:id`
* `DELETE /users/:id`
* `DELETE /users/signin`
* `OPTIONS /users`

Read more about how each endpoint works in the next section.

## Actions

All actions respond with a JSON object with a `code` attribute with the following possible string values:

* `OK` - action has completed successfully
* `BAD_REQUEST` - action has failed due to request not being in expected format
* `NOT_FOUND` - endpoint not found, or user not found
* `UNAUTHORIZED` - action requires that a user signs in first
* `FORBIDDEN` - action requires that signed-in user has permission to perform it
* `INTERNAL_ERROR` - an unexpected error has occurred while performing the action

### Create User

* `POST /users`

This endpoint creates a new user.

<table>
<tr><th>Request Body</th><th>Response</th></tr>
<tr><td>

```javascript
{
  "username": "MyUsername",
  "email": "myemail@test.com",
  "password": "abc123"
}
```

</td><td>

```javascript
{
  "code": "OK",
  "user": {
    "id": "12345",
    "username": "MyUsername",
    "email": "myemail@test.com"
  }
}
```

</td></tr>
</table>

### User Sign-in

* `POST /users/signin`

Signs a user in, starting a new session.

This endpoint sets a session cookie upon successful response, which will be sent in the next requests.

<table>
<tr><th>Request Body</th><th>Response</th></tr>
<tr><td>

```javascript
{
  "email": "myemail@test.com",
  "password": "abc123"
}
```

</td><td>

```javascript
{
  "code": "OK",
  "user": {
    "id": "12345",
    "username": "MyUsername",
    "email": "myemail@test.com"
  }
}
```

</td></tr>
</table>

### User Sign-out

* `POST /users/signout`
* `DELETE /users/signin` (alternatively)

Signs a user out, removing the session cookie.

<table>
<tr><th>Request Body</th><th>Response</th></tr>
<tr><td>(empty)</td><td>

```javascript
{
  "code": "OK"
}
```

</td></tr>
</table>

### User List

* `GET /users`

Returns a list of users with pagination. Default page size is 20.

By default, this endpoint requires the signed-in user role to be `admin`.

<table>
<tr><th>Request Query Parameters</th><th>Response</th></tr>
<tr><td>

* `/users`
  * first 20 users
* `/users?limit=10`
  * first 10 users
* `/users?start=NextPageKey000123`
  * first 20 users
  * starting from key "NextPageKey000123"

</td><td>

```javascript
{
  "code": "OK",
  "items": [
    {"id": "1", "username": "user1", "email": "email1@test.com", ...},
    {"id": "2", "username": "user2", "email": "email2@test.com", ...},
    ...
  ],
  "limit": 20,
  "next": "NextPageKey000123"
}
```

</td></tr>
</table>

The `next` attribute will be absent from the response if there are no more entries to fetch.

(Filters and sorting are not yet supported.)

### Show User

* `GET /users/:userId`

Returns data of a single user.

By default, this endpoint has the following requirements:

* user ID being requested matches with signed-in user ID; or
* the signed-in user role is `admin`.

<table>
<tr><th>Request URI</th><th>Response</th></tr>
<tr><td>

* `/users/12345`
  * shows info of user ID=12345

</td><td>

```javascript
{
  "code": "OK",
  "user": {
    "id": "12345",
    "username": "MyUsername",
    "email": "myemail@test.com"
  }
}
```

</td></tr>
</table>

### Update User

* `PUT /users/:userId`
* `PATCH /users/:userId`

Updates data of a single user, including password.

Both `PUT` and `PATCH` methods behave the same way, and partial data can be provided.

By default, this endpoint has the following requirements:

* user ID being updated matches with signed-in user ID; or
* the signed-in user role is `admin`.

(Not yet implemented) The user `role` attribute can only be updated by other admins.

<table>
<tr><th>Request Body</th><th>Response</th></tr>
<tr><td>

```javascript
{
  "username": "EditedUsername"
}
```

</td><td>

```javascript
{
  "code": "OK",
  "user": {
    "id": "12345",
    "username": "EditedUsername",
    "email": "myemail@test.com"
  }
}
```

</td></tr>
</table>

### User Delete

* `DELETE /users/:userId`

Removes a user.

By default, this endpoint has the following requirements:

* user ID being deleted matches with signed-in user ID; or
* the signed-in user role is `admin`.

If user being deleted matches the user currently signed in, the user will be automatically signed out.

<table>
<tr><th>Request Body</th><th>Response</th></tr>
<tr><td>(empty)</td><td>

```javascript
{
  "code": "OK"
}
```

</td></tr>
</table>

### Preflight Requests

* `OPTIONS /users/*`

Some clients will fire a "preflight request" prior to making the real request to verify CORS options, which are supported by this library. It returns an empty 200 status code with the appropriate headers.

### Authorization

Call `authorize` in your _other_ Google Functions to have the session cookie read from the request before calling your code:

```javascript
const users = require('google-function-authorizer')();

exports.handleRequest = function (req, res) {
  users.authorize(req, res, mainFunction);
};

function mainFunction (req, res, user) {
  // session is valid -- execute the rest of your function
}
```

If the session is valid, a `user` object will be passed along to your main function, containing a subset of stored attributes.

If the session is invalid, a `401 - Unauthorized` error will be returned automatically. If you want to take control of error processing, pass a fourth argument to `authorize` with an error function that takes `req`, `res`, and `error` arguments.

### Roles

This library comes with a very simple role management system built-in.

Users with `admin` role are able to edit and delete other users.

Users with any other role are only allowed to edit or delete themselves.

Users cannot be created with a role, since that specific endpoint is public. Another user with `admin` role must edit newly created users if they need defined roles.

## Configuration

(Not yet implemented)

With the exception of `session.secret`, all other settings have the following defaults:

```javascript
const users = require('google-function-authorizer')({
  session: {
    secret: 'MYSECRET',
    expiration: 24 * 60 * 60 * 1000, // session expiration in ms
    active: 1000 * 60 * 5 // session active duration in ms
  },
  datastore: {
    kind: 'User',
    namespace: null,
    primaryField: 'email', // affects signin endpoint
    fields: {
      username: 'username',
      email: 'email',
      password: 'password',
      role: 'role'
    }
  },
  cors: {
    allow: '*'
  },
  rules: {
    // The following values are acceptable for the fields below:
    // - true: public and unrestricted access
    // - "user": access for logged-in users only
    // - "self": access for record owner only, or admins
    // - "admin": admin access only
    show: 'self',
    list: 'admin',
    delete: 'self'
  }
});

exports.handleRequest = function (req, res) {
  users.handle(req, res);
};
```

__Configuration must be the same across all functions__, so if you change any of the default values, make sure to replicate them accordingly.

_When Google Cloud Functions support environment variables, we will change this approach so configuration can be unified and the copy/pasting avoided._

## TODO/Wishlist

* Support custom models and validation.
* Email service support (Mailgun, SendGrid, etc) for sending various confirmation messages.
* In-memory caching support for `authorization`.
* Allow `authorization` caching to be customized/disabled.
* Allow CORS to be configured.
* Allow configuration of Datastore options, such as kind, namespace, and attribute names.
* Allow configuration of client-session options.
* Support other data stores (like MySQL).
* Support other password hashing libraries.
* Support JSON Web Tokens instead of client-sessions.

__This is a work in progress.__
