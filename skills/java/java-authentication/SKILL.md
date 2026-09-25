---
name: 'java-authentication'
description: 'Set credentials on the PayPal Server SDK Java SDK. Load before configuring any scheme, or when a call comes back 401 or 403. The builder won''t tell you each credential field is pre-initialised with empty strings so a forgotten setter fails inside the SDK, that only client credentials fetches a token for you, or which getter reads the model back.'
---

# Authenticating an APIMatic Java SDK client
> **This API declares at least one security scheme**, so everything below applies to this SDK:
> `<root>/authentication/` exists and `{Api}Client.Builder` carries a credential setter per scheme. Read
> those setters for the real names.

How you authenticate depends on the security scheme(s) the API uses. APIMatic surfaces each scheme as a
**`{Scheme}Model` data class plus a matching setter on the client `Builder`**. Set the ones your API uses
when you build the client (see `java-client-initialization`).

To see which schemes a specific SDK accepts, read the **credential setters on `{Api}Client.Builder`** —
those are the source of truth. The models live in `<root>/authentication/`; the credentials *interface*
sits in the root package when the API has a single scheme and in `<root>/authentication/` when it has
several. `doc/auth/*.md` lists the same set in readable form.

## The universal shape

Every scheme follows the same three steps: build the model, pass it to the client builder, forget about
it.

```java
import {rootPackage}.{Api}Client;
import {rootPackage}.authentication.{Scheme}Model;

{Api}Client client = new {Api}Client.Builder()
        .{scheme}Credentials(new {Scheme}Model.Builder(
                        System.getenv("API_CLIENT_ID"),      // required credentials: Builder constructor
                        System.getenv("API_CLIENT_SECRET"))
                .build())
        .build();
```

- **Required credentials are constructor arguments** on the model's `Builder`, and each throws
  `NullPointerException` on `null` — so a missing environment variable fails at construction, not on the
  first call.
- **Optional credentials are fluent setters** on the same `Builder`.
- The model's own constructor is private; `build()` is the only way to make one. `model.toBuilder()`
  gives you a pre-populated `Builder` to derive a variant from.

## Basic auth

```java
.basicAuthCredentials(new BasicAuthModel.Builder(
                System.getenv("API_USERNAME"),
                System.getenv("API_PASSWORD"))
        .build())
```

## API key in a header

```java
.customHeaderAuthenticationCredentials(new CustomHeaderAuthenticationModel.Builder(
                System.getenv("API_KEY"))
        .build())
```

The header name is fixed by the generated scheme — you supply only the value. A scheme with several
parameters takes several constructor arguments, in the order the model's `Builder` declares them.

## API key in a query parameter

```java
.customQueryAuthenticationCredentials(new CustomQueryAuthenticationModel.Builder(
                System.getenv("API_KEY"))
        .build())
```

## Bearer / access token

```java
.bearerAuthCredentials(new BearerAuthModel.Builder(
                System.getenv("API_ACCESS_TOKEN"))
        .build())
```

## OAuth 2.0 — client credentials

This is the only grant the SDK drives end to end: it fetches a token on the first call that needs one,
caches it, and refetches when it expires.

```java
.clientCredentialsAuth(new ClientCredentialsAuthModel.Builder(
                System.getenv("API_OAUTH_CLIENT_ID"),
                System.getenv("API_OAUTH_CLIENT_SECRET"))
        .build())
```

The other grants (authorization code, resource-owner password) do **not** fetch anything on their own —
you drive the flow and rebuild the client with the token; see *More schemes* below.

## Reading credentials back off the client

Each scheme produces two getters on the client:

- `get{Scheme}Credentials()` — the credentials *interface*, backed by the auth manager. This is where the
  OAuth methods (`fetchToken`, `isTokenExpired`, …) live.
- `get{Scheme}Model()` — the model you supplied, so you can `toBuilder()` a variant off it.

For OAuth grants in a single-scheme SDK the interface drops the `Credentials` suffix, so the getter is
`getClientCredentialsAuth()` / `getAuthorizationCodeAuth()` rather than `get...Credentials()`. Confirm
the exact names in `{Api}Client.java`.

## More schemes

**Authorization code** is not automatic: send the user to
`client.get{Scheme}Auth().buildAuthorizationUrl()`, exchange the code your redirect endpoint receives
with `fetchToken(code)`, then **rebuild the client** with that token through
`client.newBuilder().{scheme}Auth(client.get{Scheme}AuthModel().toBuilder().oAuthToken(token).build())`.
Until the token is attached, calls fail with an auth error saying an OAuth token is needed — that is the
symptom of skipping the rebuild. The credentials interface also carries `refreshToken()` and
`isTokenExpired()`. **Resource-owner password** is the same flow without the browser step; its model
`Builder` takes client id, client secret, username and password.

Only client credentials refreshes itself, through `oAuthOnTokenUpdate` (persist) and `oAuthTokenProvider`
(load); for the other grants call `refreshToken()` yourself and rebuild. The token model is an ordinary
Jackson-serializable class, so storing it as JSON is enough.

There is **no combined credentials object**: set every scheme the operations you call require, and the
controller applies the composition per operation — **AND** applies each scheme in the group, **OR** uses
the first satisfied one. The `.withAuth(...)` block in the controller method is what decides.

To see what this SDK accepts, list the `Builder` methods on `{Api}Client.java` ending in `Credentials`
(or the suffix-less OAuth ones), then read the matching `{Scheme}Model` `Builder` constructor under
`<root>/authentication/` for the required values; `doc/auth/*.md` restates it with a snippet.

## Notes

- **A forgotten credential setter fails inside the SDK — you never see the API's `401`.** Every
  credential field on the client `Builder` is initialized to a model built with **empty strings** for its
  required values. For a header/query/basic/bearer scheme an empty value fails auth validation *before*
  the request is built and nothing leaves the process. An OAuth 2 grant validates on the **token**
  instead: authorization-code and resource-owner-password fail locally the same way, but
  **client-credentials first attempts a token fetch** — a real POST to the token endpoint with the blank
  id/secret — and swallows its `ApiException`/`IOException` inside `getTokenFromProvider()`, leaving the
  token `null`. Either way the call then throws the unchecked `AuthValidationException`
  (`io.apimatic.core.exceptions`); it extends `RuntimeException`, so the
  `catch (ApiException | IOException)` ladder from **java-error-handling** does not cover it. Set every
  scheme the endpoints you call require.
- Set credentials when you build the client. To rotate them later, derive a new client with
  `client.newBuilder().{scheme}Credentials(...).build()` — the built client is immutable.

## Next

- Step 3, make your first call → **java-calling-endpoints**
- A call that comes back `401` or `403` → **java-error-handling**
