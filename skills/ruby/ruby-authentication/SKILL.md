---
name: 'ruby-authentication'
description: 'Set credentials on the PayPal Server SDK Ruby SDK. Load before configuring any scheme, or when a call comes back 401 or 403. The argument list won''t tell you each scheme takes a constructed credentials object passed by name, that OAuth identifiers keep fixed spellings whatever the scheme is called, or which schemes fetch a token for you.'
---

# Authenticating an APIMatic Ruby SDK client

How you authenticate depends on the security scheme(s) the API uses. APIMatic surfaces **one named
parameter per scheme on `Client.new`**, and — for every scheme except custom authentication — **an
immutable credentials class to pass to it** (see `ruby-client-initialization`).

> **Custom authentication is the exception.** Its file holds only the scheme class (e.g.
> `class CustomAuth < CoreLibrary::HeaderAuth`), built from the whole configuration and shipped as a
> stub for the SDK's maintainer to complete. There is **no `{SchemeCredentials}` class to build**, even
> though the `{scheme}_credentials:` parameter still exists. Check the scheme's file for a credentials
> class before writing the call — a custom scheme means the SDK has to be completed or regenerated, not
> configured from your application.

## Finding which schemes this SDK accepts

Three places, in order of directness:

1. `lib/paypal_server_sdk/configuration.rb` — the credential parameters on `Configuration#initialize`.
   **These are the source of truth**; the same parameters appear on `Client#initialize`.
2. `lib/paypal_server_sdk/http/auth/` — one file per scheme, each holding the scheme class and, for every
   scheme but custom authentication, its `{SchemeCredentials}` data class. The credentials class is a
   **sibling** of the scheme class in the same module, not nested inside it, so it is referenced as
   `PaypalServerSdk::{SchemeCredentials}`. A file with only a scheme class is a custom-auth stub — there
   is nothing to build for it.
3. `README.md` / `doc/auth/` — the generated authentication section, which names each scheme and lists
   its parameters and their types.

## The shape: build a credentials object, pass it by name

```ruby
require 'paypal_server_sdk'

client = PaypalServerSdk::Client.new(
  {scheme}_credentials: PaypalServerSdk::{SchemeCredentials}.new(
    {param}: ENV.fetch('MY_API_PARAM'),
    {other_param}: ENV.fetch('MY_OTHER_PARAM')
  )
)
```

Every generated credentials class has the same shape:

- **`initialize`** takes **keyword arguments**, one per auth parameter. A required parameter has no
  default and is checked immediately — `raise ArgumentError, '{param} cannot be nil' if {param}.nil?` —
  so a missing credential fails at construction, not on the first call. Optional parameters default to
  `nil` (or the value the spec declares).
- **`attr_reader`** for each parameter, and nothing else: the object is **immutable**.
- **`clone_with(...)`** returns a new instance with the arguments you pass replacing the current values.
  It fills each missing argument with `||=`, so it cannot clear a value back to `nil`/`false`.
- **`self.from_env`** builds the object from `UPPER_SNAKE` environment variables and returns `nil` when
  every required variable is unset. On an API with a **single** scheme the variable is named after the
  parameter (`{PARAM}`); with **multiple** schemes it is prefixed with the scheme (`{SCHEME}_{PARAM}`).
  Grep `from_env` in the scheme's file for the exact names.

The credentials you passed are readable back as `client.config.{scheme}_credentials`.

## Basic auth

```ruby
client = PaypalServerSdk::Client.new(
  {basic_auth_credentials}: PaypalServerSdk::{BasicAuthCredentials}.new(
    {username_param}: ENV.fetch('API_USERNAME'),
    {password_param}: ENV.fetch('API_PASSWORD')
  )
)
```

The scheme class Base64-encodes the two values into an `Authorization: Basic ...` header for you.

## API key — custom header or custom query parameter

An API key scheme is generated as either a header scheme or a query scheme; the header/parameter name is
fixed by the generated class, so all you supply is the value:

```ruby
client = PaypalServerSdk::Client.new(
  {api_key_credentials}: PaypalServerSdk::{ApiKeyCredentials}.new(
    {api_key_param}: ENV.fetch('API_KEY')
  )
)
```

Open the scheme's file under `lib/paypal_server_sdk/http/auth/` to see whether it subclasses the header or
the query auth base and which name it sends the key under.

## OAuth 2.0 — bearer token

When the API declares a bearer token you already hold, it is just another credentials parameter:

```ruby
client = PaypalServerSdk::Client.new(
  {o_auth_credentials}: PaypalServerSdk::{OAuthCredentials}.new(
    access_token: ENV.fetch('ACCESS_TOKEN')
  )
)
```

A bearer-token scheme's keyword is `access_token:`. Every other grant names its credentials
`o_auth_`-prefixed — `o_auth_client_id:`, `o_auth_client_secret:`, `o_auth_token:`, `o_auth_scopes:` — in
**every** build: the generator derives them from fixed constants, so there is no `oauth_client_id:`
spelling and no casing convention an SDK is built with changes them. (The auth-manager reader on the
client is underscore-cased from the API's own scheme name instead — a scheme the API calls
`Oauth2` reads `client.oauth2`, one it calls `OAuthCCG` reads `client.o_auth_ccg`; grep `@auth_managers[`
in `client.rb` for the real reader.) The class name and the `Client.new` parameter *are* per-SDK, so
confirm those, and the parameter list, against the credentials class's `initialize` in
`lib/paypal_server_sdk/http/auth/` or the *Getter* column of `doc/auth/*.md` — a wrong keyword raises
`ArgumentError: unknown keyword`.

## OAuth 2.0 — client credentials

Constructing the client with the client id and secret is **enough**. The auth manager fetches a token
lazily on the first call that needs one, holds it, and re-fetches when it expires — nothing to attach:

```ruby
client = PaypalServerSdk::Client.new(
  {o_auth_credentials}: PaypalServerSdk::{OAuthCredentials}.new(
    o_auth_client_id: ENV.fetch('O_AUTH_CLIENT_ID'),
    o_auth_client_secret: ENV.fetch('O_AUTH_CLIENT_SECRET')
  )
)

result = client.{controller_name}.{operation}   # the token is fetched here, on demand
```

Confirm it for your SDK before relying on it: the grant's class under `lib/paypal_server_sdk/http/auth/`
has a `valid` method that assigns the token it fetched, and `doc/auth/` says so in prose. A **bearer-token
scheme has no token endpoint**, so there is nothing to auto-fetch — you supply the token yourself.

Fetch a token explicitly only to pre-warm one, inspect it, or reuse one you persisted. A token you fetch
this way is **not** written back into the credentials object, which is immutable — so attaching it means
cloning the credentials, cloning the configuration, and rebuilding the client:

```ruby
begin
  token = client.{auth_key}.fetch_token

  credentials = client.config.{o_auth_credentials}.clone_with(o_auth_token: token)
  config = client.config.clone_with({o_auth_credentials}: credentials)
  client = PaypalServerSdk::Client.new(config: config)
rescue PaypalServerSdk::APIException => e
  # handle the failed token request
end
```

`{auth_key}` is a reader the client exposes for each OAuth scheme. Grep `@auth_managers[` in
`lib/paypal_server_sdk/client.rb` — the reader you call is the **method whose body returns that entry**,
not the camelCase hash key it looks up. The manager also exposes `token_expired?(token)`, which takes the
`OAuthToken` you hold (from `fetch_token`, or from your own store) as a **required argument** — it does not
inspect the manager's stored token, and no public reader exposes that one — and, when the grant issues
refresh tokens, `refresh_token(additional_params: nil)`. A token from either is re-attached with the same
clone-and-rebuild sequence above.

A client-credentials scheme's credentials class additionally accepts:

| Parameter | Type | Purpose |
| --- | --- | --- |
| `o_auth_token_provider` | `proc { \|OAuthToken, OAuth2\| }` | callback the SDK invokes to obtain/refresh the token |
| `o_auth_on_token_update` | `proc { \|OAuthToken\| }` | callback fired when the token is updated |
| `o_auth_clock_skew` | `Integer` | seconds of skew allowed when checking token expiry |

To make a token survive a restart, **verify the update callback actually fires** in your SDK before
building persistence on it. The dependable route is to save the token `fetch_token` returned to you and
pass it back into the credentials object on the next boot.

## More schemes

**Authorization code** is two calls with your redirect handling in between: send the user to
`client.{auth_key}.get_authorization_url(state: nil, additional_params: nil)`, then
`client.{auth_key}.fetch_token(authorization_code)` — the code is positional — and rebuild the client
through `client.config.clone_with(...)` with `o_auth_token:` set on the cloned credentials. That grant's
credentials class also carries `o_auth_redirect_uri`, and an implicit-grant scheme has
`get_authorization_url` but **no** `fetch_token`. **Resource-owner password** prefixes the user
credentials: `o_auth_username:` / `o_auth_password:`, not the bare names a Basic scheme uses.

Where the API declares more than one scheme, each has its own credentials class and its own named
parameter on `Client.new`; set every one the operations you call require and the SDK applies the
combination each operation declares. With multiple schemes, `from_env` reads **scheme-prefixed**
variables (`{SCHEME}_{PARAM}`) rather than bare ones.

A scheme the API declares as **custom** generates only the scheme class, whose `error_message` and
`valid` are TODO stubs, and **no `{SchemeCredentials}` class** — naming one raises `NameError` even
though the `{scheme}_credentials:` parameter still exists. A *custom header* scheme is an ordinary
API-key scheme and does generate its credentials class, so read the file rather than its name.

Some older single-scheme SDKs also accept each auth parameter flat on `Client.new` and print a
deprecation warning; use the credentials object.

## Notes

- A given SDK only exposes the credential parameters for the schemes its API uses; those names are
  generated per API, hence the `{...}` placeholders above.
- Credentials are set when the client is constructed. There is no setter — changing them means a new
  client (via `config.clone_with`).
- `Client.from_env` reads the credentials out of `ENV` for you.
- Each scheme class defines `error_message`, the string reported when the scheme cannot be satisfied
  (e.g. a nil credential, or an expired OAuth token). If a call fails on authentication, that message
  names the parameter that is missing.

## Next

- Step 3, make your first call → **ruby-calling-endpoints**
- A call that comes back `401` or `403` → **ruby-error-handling**
