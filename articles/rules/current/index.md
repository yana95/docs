---
description: Learn what Rules are and how you can use them to customize and extend Auth0's capabilities.
toc: true
---

# Rules

**Rules** are functions written in JavaScript that are executed in Auth0 as part of the transaction every time a user authenticates to your application. They are executed after the authentication and before the authorization.

Rules allow you to easily customize and extend Auth0's capabilities. They can be chained together for modular coding and can be turned on and off individually.

![](/media/articles/rules/flow.png)

1. An app initiates an authentication request to Auth0.
1. Auth0 routes the request to an Identity Provider through a configured connection.
1. The user authenticates successfully.
1. The tokens ([id_token](/tokens/id-token) and/or [access_token](/tokens/access-token)) pass through the Rules pipeline, and are sent to the app.

Among many possibilities, Rules can be used to:

* __Profile enrichment__: query for information on the user from a database/API, and add it to the user profile object.
* Create __authorization rules__ based on complex logic (anything that can be written in JavaScript).
* __Normalize attributes__ from different providers beyond what is provided by Auth0.
* Reuse information from existing databases or APIs for migration scenarios.
* Keep a __white-list of users__ and deny access based on email.
* __Notify__ other systems through an API when a login happens in real-time.
* Enable counters or persist other information. For information on storing user data, see: [Metadata in Rules](/rules/metadata-in-rules).
* Enable __multifactor__ authentication, based on context (e.g. last login, IP address of the user, location, etc.).
* Modify tokens: Change the returned __scopes__ of the `access_token` and/or add claims to it, and to the `id_token`.

## Video: Using Rules
Watch this video learn all about rules in just a few minutes.

<%= include('../../videos/_video', { id: 'g7dy1fpwc3' }) %>

## Rule Syntax

A Rule is a function with the following arguments:

* `user`: the user object as it comes from the identity provider. For a complete list of the user properties, see [User Profile Structure](/user-profile/user-profile-structure).

* `context`: an object containing contextual information of the current authentication transaction, such as user's IP address, application, location. For a complete list of context properties, see [Context Argument Properties in Rules](/rules/context).

* `callback`: a function to send back potentially modified tokens back to Auth0, or an error. Because of the async nature of Node.js, it is important to always call the `callback` function, or else the script will timeout.

## Examples

To create a Rule, or try the examples below, go to [New Rule](${manage_url}/#/rules/create) in the Rule Editor on the dashboard.

### Hello World

This rule will add a `hello` claim (with the value `world`) to the `id_token` that will be afterwards sent to the application.

```js
function (user, context, callback) {
  context.idToken["http://mynamespace/hello"] = "world";
  console.log('===> set "hello" for ' + user.name);
  callback(null, user, context);
}
```

Note that the claim is namespaced: we named it `http://mynamespace/hello` instead of just `hello`. This is what you have to do in order to add arbitrary claims to an `id_token` or `access_token`.

Any non-Auth0 HTTP or HTTPS URL can be used as a namespace identifier, and any number of namespaces can be used. The namespace URL does not have to point to an actual resource, it’s only used as an identifier and will not be called by Auth0. For more information refer to [User profile claims and scope](/api-auth/tutorials/adoption/scope-custom-claims).

<div class="alert alert-info">
  You can add <code>console.log</code> lines for <a href="#debugging">debugging</a> or use the <a href="/extensions/realtime-webtask-logs">Real-time Webtask Logs Extension</a>.
</div>


### Add roles to a user

In this example, all authenticated users will get a **guest** role, but `johnfoo@gmail.com` will also be an **admin**:

```js
function (user, context, callback) {
  if (user.email === 'johnfoo@gmail.com') {
    context.idToken["http://mynamespace/roles"] = ['admin', 'guest'];
  }else{
    context.idToken["http://mynamespace/roles"] = ['guest'];
  }

  callback(null, user, context);
}
```

At the beginning of the rules pipeline, John's `context` object will be:

```json
{
  "clientID": "YOUR_CLIENT_ID",
  "clientName": "YOUR_CLIENT_NAME",
  "clientMetadata": {},
  "connection": "YOUR_CONNECTION_NAME",
  "connectionStrategy": "auth0",
  "protocol": "oidc-implicit-profile",
  "accessToken": {},
  "idToken": {},
  //... other properties ...
}
```

After the rule executes, the `context` object will have the added namespaced claim as part of the `id_token`:

```json
{
  "clientID": "YOUR_CLIENT_ID",
  "clientName": "YOUR_CLIENT_NAME",
  "clientMetadata": {},
  "connection": "YOUR_CONNECTION_NAME",
  "connectionStrategy": "auth0",
  "protocol": "oidc-implicit-profile",
  "accessToken": {},
  "idToken": { "http://mynamespace/roles": [ "admin", "guest" ] },
  //... other properties ...
}
```

When your application receives the `id_token`, it will verify and decode it, in order to access this added custom claim. The payload of the decoded `id_token` will be similar to the following sample:

```json
{
  "iss": "https://${account.namespace}/",
  "sub": "auth0|USER_ID",
  "aud": "YOUR_CLIENT_ID",
  "exp": 1490226805,
  "iat": 1490190805,
  "nonce": "...",
  "at_hash": "...",
  "http://mynamespace/roles": [
    "admin",
    "guest"
  ]
}
```

For more information on the `id_token` refer to [ID Token](/tokens/id-token).

<div class="alert alert-info">
Properties added in a rule are <strong>not persisted</strong> in the Auth0 user store. Persisting properties requires calling the Auth0 Management API.
</div>

### Deny access based on a condition

In addition to adding claims to the `id_token`, you can return an *access denied* error.

```js
function (user, context, callback) {
  if (context.clientID === "BANNED_CLIENT_ID") {
    return callback(new UnauthorizedError('Access to this application has been temporarily revoked'));
  }

  callback(null, user, context);
}
```

This will cause a redirect to your callback url with an `error` querystring parameter containing the message you set. (e.g.: `https://yourapp.com/callback?error=unauthorized&error_description=Access%20to%20this%20application%20has%20been%20temporarily%20revoked`). Make sure to call the callback with an instance of `UnauthorizedError` (not `Error`).

<div class="alert alert-info">
  Error reporting to the app depends on the protocol. OpenID Connect apps will receive the error in the querystring. SAML apps will receive the error in a <code>SAMLResponse</code>.
</div>

### Copy User Metadata to ID Token

This will read the `favorite_color` user metadata, and add it as a namespaced claim at the `id_token`.

```js
function(user, context, callback) {

  // copy user metadata value in id_token
  context.idToken['http://fiz/favorite_color'] = user.favorite_color;

  callback(null, user, context);
}
```

### API Authorization: Modify Scope

This will override the returned scopes of the `access_token`. The rule will run after user authentication and before authorization.

```js
function(user, context, callback) {

  // change scope
  context.accessToken.scope = ['array', 'of', 'strings'];

  callback(null, user, context);
}
```

The user will be granted three scopes: `array`, `of`, and `strings`.

### API Authorization: Add Claims to Access Tokens

This will add one custom namespaced claim at the `access_token`.

```js
function(user, context, callback) {

  // add custom claims to access token
  context.accessToken['http://foo/bar'] = 'value';

  callback(null, user, context);
}
```

After this rule executes, the `access_token` will contain one additional namespaced claim: `http://foo/bar=value`.

## Create Rules with the Management API

Rules can also be created by creating a POST request to `/api/v2/rules` using the [Management APIv2](/api/management/v2#!/Rules/post_rules).

This will creates a new rule according to the following input arguments:

- **name**: The name of the rule. It can only contain alphanumeric characters, spaces and '-', and cannot start nor end with '-' or spaces.

- **script** : Τhe script that contains the rule's code. This is the same as what you would enter when creating a new rule using the [dashboard](${manage_url}/#/rules/create).

- **order**: This field is optional and contains a `number`. This number represents the rule's order in relation to other rules. A rule with a lower order than another rule executes first. If no order is provided it will automatically be one greater than the current maximum.

- **enabled**: This field can contain an optional `boolean`. If `true`, the rule will be enabled, if it's `false` it will be disabled.

Example of a body schema:

```
{
  "name": "my-rule",
  "script": "function (user, context, callback) {\n  callback(null, user, context);\n}",
  "order": 2,
  "enabled": true
}
```
Use this to create the POST request:

```har
{
  "method": "POST",
  "url": "https://${account.namespace}/api/v2/rules",
  "headers": [{
    "name": "Content-Type",
    "value": "application/json"
  }],
  "postData": {
    "mimeType": "application/json",
    "text": "{\"name\":\"my-rule\",\"script\":\"function (user, context, callback) {callback(null, user, context);}\",\"order\":2,\"enabled\":true}"
  }
}
```

<div class="alert alert-info">
You can use the <a href="https://www.npmjs.com/package/auth0-custom-db-testharness">auth0-custom-db-testharness library</a> to deploy, execute, and test the output of Custom DB Scripts using a Webtask sandbox environment.
</div>

## How to Debug Rules

You can add `console.log` lines in the rule's code for debugging. The [Rule Editor](${manage_url}/#/rules/create)  provides two ways for seeing the output:

![](/media/articles/rules/rule-editor.png)

1. **TRY THIS RULE**: opens a pop-up where you can run a rule in isolation. The tool provides a mock **user** and **context** objects. Clicking **TRY** will result on the the Rule being run with those two objects as input. `console.log` output will be displayed too.

![](/media/articles/rules/try-rule.png)

2. **REALTIME LOGS**: an [extension](${manage_url}/#/extensions) that displays all logs in real-time for all custom code in your account. This includes all `console.log` output, and exceptions.

3. **DEBUG RULE**: similar to the above, displays instructions for installing, configuring and running the [webtask CLI](https://github.com/auth0/wt-cli) for debugging rules. Paste these commands into a terminal to see the `console.log` output and any unhandled exceptions that occur during Rule execution.

  For example:

  ```sh
  ~  npm install -g wt-cli
  ~  wt init --container "youraccount" --url "https://sandbox.it.auth0.com" --token "eyJhbGci...WMPGI" -p "youraccount-default-logs"
  ~  wt logs -p "youraccount-default-logs"
  [18:45:38.179Z]  INFO wt: connected to streaming logs (container=youraccount)
  [18:47:37.954Z]  INFO wt: webtask container assigned
  [18:47:38.167Z]  INFO wt: ---- checking email_verified for some-user@mail.com! ----
  ```

  This debugging method works for rules tried from the dashboard and those actually running during user authentication.

## Cache expensive resources

The code sandbox Rules run on allows storing _expensive_ resources that will survive individual execution.

This example, shows how to use the `global` object to keep a mongodb connection:

```js

  ...

  //If the db object is there, use it.
  if (global.db){
    return query(global.db, callback);
  }

  //If not, get the db (mongodb in this case)
  mongo('mongodb://user:pass@mymongoserver.com/my-db',  function (db){
    global.db = db;
    return query(db, callback);
  });

  //Do the actual work
  function query(db, cb){
    //Do something with db
    ...
  });

  ...

```

Notice that the code sandbox in which Rules run on, can be recycled at any time. So your code __must__ always check `global` to contain what you expect.

## Available modules

For security reasons, the Rules code runs in a JavaScript sandbox based on [webtask.io](https://webtask.io) where you can use the full power of the ECMAScript 5 language.

For a list of currently supported sandbox modules, see: [Modules Supported by the Sandbox](https://tehsis.github.io/webtaskio-canirequire).

## Further reading

* [Redirecting users from within rules](/rules/redirect)
