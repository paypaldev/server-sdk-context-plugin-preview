---
name: 'php-authentication'
description: 'Set credentials on the PayPal Server SDK PHP SDK. Load before configuring any scheme, or when a call comes back 401 or 403. The builder won''t tell you each scheme has its own credentials builder, that a single-scheme OAuth SDK drops the `Credentials` suffix from the accessor, or that the token provider lives on the credentials builder rather than the client.'
---

# Authenticating an APIMatic PHP SDK client
Which schemes exist is decided by the API spec. **The authoritative list is the set of
`->{scheme}Credentials(…)` setters on `src/{Client}Builder.php`** (mirrored on
`src/ConfigurationInterface.php` as one `get{Scheme}CredentialsBuilder()` plus one credentials getter per
scheme, and documented one-file-per-scheme under `doc/auth/`). Start there — and note that only the
*setter* name is uniform: the credentials getter is `get{Scheme}Credentials()`, **except** when the API has
exactly one scheme and it is an OAuth 2 grant, where the `Credentials` suffix is dropped and it is
`get{Scheme}()`. Grepping only for the suffixed getter on such an SDK finds nothing and looks like the
scheme is absent.

## The shape: one credentials builder per scheme

```php
use PaypalServerSdkLib\{Client}Builder;
use PaypalServerSdkLib\Authentication\{Scheme}CredentialsBuilder;

$client = {Client}Builder::init()
    ->{scheme}Credentials(
        {Scheme}CredentialsBuilder::init(/* required params, positionally */)
            // optional params are fluent setters
    )
    ->build();
```

Every credentials builder follows the same rules:

- `init(...)` takes the scheme's **required** parameters, positionally.
- Optional parameters are fluent setters returning `$this`.
- The builder is passed to the client builder, which merges its config — you never call
  `getConfiguration()` yourself.

**Where `{scheme}` comes from depends on how many schemes the API has.** With more than one, each name is
the spec's own scheme name. With exactly one, the generator **discards** the spec's name and uses a fixed
name for the auth *type* (`BasicAuth`, `BearerAuth`, `ClientCredentialsAuth`,
`CustomHeaderAuthentication`, …) — that is the default, though a single-scheme SDK can also be built to
keep the API's own scheme name. So `->basicAuthCredentials(…)`, `->apiKeyCredentials(…)` and
`->oAuthCCGCredentials(…)` are what one *multi-scheme* spec produced; yours may differ. Only the *pattern*
`->{schemeName}Credentials(…)` holds — read the setters off `src/{Client}Builder.php`.

### Where the classes live

Credentials **builders** and auth **managers** are always in `PaypalServerSdkLib\Authentication`. The
credentials *interface* moves: `PaypalServerSdkLib\Authentication` when the API has more than one scheme,
but the **root** `PaypalServerSdkLib` namespace when it has exactly one. Its **name** moves too — it is
`{Scheme}Credentials`, except on an API whose single scheme is an OAuth 2 grant, where the `Credentials`
suffix is dropped and the interface is just `{Scheme}` (e.g. `ClientCredentialsAuth`). Let your editor or
a grep of `src/` settle the `use` statement rather than assuming.

## Basic auth

```php
$client = {Client}Builder::init()
    ->{scheme}Credentials(
        {Scheme}CredentialsBuilder::init(
            getenv('API_USERNAME'),
            getenv('API_PASSWORD')
        )
    )
    ->build();
```

Sends `Authorization: Basic base64(username:password)`.

## Bearer token

```php
->{scheme}Credentials({Scheme}CredentialsBuilder::init(getenv('API_ACCESS_TOKEN')))
```

Sends `Authorization: Bearer <token>`.

## API key — header or query parameter

Placement (header vs query) and the wire name are fixed by the generated manager; you only supply the
value(s). An API key scheme may take **more than one** parameter — read `init(...)`'s signature.

```php
->{scheme}Credentials({Scheme}CredentialsBuilder::init(getenv('API_TOKEN'), getenv('API_KEY')))
```

## OAuth 2

All grants share `oAuthToken(?OAuthToken $token)` and `oAuthClockSkew(int $seconds)` on the credentials
builder, and `isTokenExpired(?OAuthToken $token = null)` on the manager.

`OAuthToken` itself lives in `src/Models/OAuthToken.php`, with a matching `OAuthTokenBuilder` under
`Models\Builders`. Its accessors are `getAccessToken()`, `getTokenType()`, `getExpiresIn()`,
`getScope()`, `getRefreshToken()` and `getExpiry()` / `setExpiry()` — the ones to read when you persist
a token and rebuild it later.

> **The prefix is `oAuth`, with a capital A — not `oauth`.** Every generated setter is spelled
> `oAuthToken`, `oAuthScopes`, `oAuthClockSkew`, `oAuthTokenProvider`, `oAuthOnTokenUpdate`, and the
> token model is the class `OAuthToken`. PHP resolves *method* calls case-insensitively so a mis-cased
> setter happens to work, but a grep of `src/Authentication/` for the lowercase spelling finds nothing, a
> static analyser flags it, and a mis-cased **class** name in a `use` statement is a hard PSR-4 autoload
> failure. Copy the spelling from the source.

**What differs between grants is whether the SDK acquires the token for you:**

| Grant | `init(...)` takes | Token acquisition |
| --- | --- | --- |
| Client credentials | client id, client secret | **automatic** — fetched and refreshed on demand |
| Resource-owner password | client id, client secret, username, password | **automatic** |
| Authorization code | client id, client secret, redirect URI | **you drive it** — build the consent URL, then `fetchToken($code)` |
| Implicit | client id | **you drive it** — `buildAuthorizationUrl()` only; the manager has no `fetchToken` |

**`oAuthScopes([...])` is generated only where the spec declares scopes for that grant, and in practice
that is usually the authorization-code builder alone** — a client-credentials or resource-owner-password
builder frequently has no scopes setter at all, and calling one that was never generated is a fatal
`Call to undefined method`. Check the builder in `src/Authentication/` before adding the line. Scopes are
validated
against a generated scope class in `src/Models/`: a class of `public const` strings with a static
`checkValue()`. Its name is `OAuthScope` on a single-scheme API and `OAuthScope{Scheme}` when the API has
more than one scheme, where `{Scheme}` is that scheme's name spelled exactly as `src/Authentication/`
spells it — it comes from the spec, so its casing is whatever the spec gave it (e.g.
`OAuthScopeOAuthACG`), plus any enum postfix the build adds. Don't guess — follow the `use`
statement at the top of the credentials builder, or grep `src/Models/` for `checkValue`. Pass its
constants, not raw strings. If an operation's `doc/controllers/*.md` section has a *Requires scope*
block, omitting the scope is a 401/403 at call time, not a build error.

### Client credentials (and resource-owner password) — automatic

```php
$client = {Client}Builder::init()
    ->{scheme}Credentials(
        {Scheme}CredentialsBuilder::init(getenv('OAUTH_CLIENT_ID'), getenv('OAUTH_CLIENT_SECRET'))
            ->oAuthScopes([{ScopeClass}::{CONSTANT}])   // only if this builder has the setter
            ->oAuthOnTokenUpdate(function (OAuthToken $token): void {
                // persist $token so a restart doesn't re-authorize
            })
            ->oAuthTokenProvider(function (?OAuthToken $last, $manager): OAuthToken {
                // supply a stored token, or return $manager->fetchToken()
            })
    )
    ->build();
```

The manager fetches a token before the first call and re-fetches when the cached one is expired. Both
callbacks are optional; `oAuthOnTokenUpdate` is how you persist a token, `oAuthTokenProvider` how you
supply one you already have. **Neither is available on the authorization-code or implicit grants.**

### Authorization code — you drive the flow

```php
$authUrl = $client->get{Scheme}()->buildAuthorizationUrl();   // send the user here
// …after the redirect comes back with ?code=…
$token = $client->get{Scheme}()->fetchToken($_GET['code']);
```

Until a token is present, calls fail with `\InvalidArgumentException` ("Client is not authorized. An
OAuth token is needed to make API calls.") — **not** `ApiException`. Refresh with
`$client->get{Scheme}()->refreshToken()`.

**`fetchToken()` itself raises `\InvalidArgumentException` too**, with a third message again: the
provider's error payload serialized into the string. So every failure on this path arrives as the same
framework exception, separated only by which message it carries, and **none of them is an
`ApiException`** — a catch ladder built on the SDK's own base class misses all three.
A typed OAuth exception class may well be generated and registered for the credential-failure statuses,
and **where `src/Http/ApiResponse.php` exists it is unreachable**: the token operation's handler chain
ends `->returnApiResponse()`, so the runtime returns the wrapper before it reaches the throw that would
construct that class. The obvious `catch` compiles, reads correctly, and never runs. That is the same
filesystem check **php-error-handling** opens with, applied to the token endpoint — read the handler
chain rather than the class list, there and here.

Note the accessor name: for an API with a **single** OAuth scheme it is `get{Scheme}()`, without the
`Credentials` suffix; with multiple schemes it is `get{Scheme}Credentials()`. Read
`src/ConfigurationInterface.php` rather than guessing.

### Reattaching a stored token

A built client is immutable, so a token you loaded from storage is applied by **rebuilding**:

```php
$client = $client
    ->toBuilder()
    ->{scheme}Credentials($client->get{Scheme}CredentialsBuilder()->oAuthToken($token))
    ->build();
```

`get{Scheme}CredentialsBuilder()` returns `null` when the scheme's required parameters were never set —
guard it if the credentials are optional in your configuration.

## Custom auth

A custom scheme has a `{Scheme}Manager` (so
`CustomAuthenticationManager` under the single-scheme fallback name, not `CustomAuthManager`) whose
`apply()` body is a `// TODO: Add your custom authentication here` stub — the signing is hand-written into
the SDK, never configured from your application. **A credentials builder and a `->{scheme}Credentials(…)`
setter are still generated when the scheme declares parameters**, and their values reach `apply()` through
the manager's `get{Param}()` getters; a parameterless custom scheme gets neither and there is genuinely
nothing to set. Read `src/{Client}Builder.php` before you either invent a setter or declare there is none.

## More schemes

**Resource-owner password** takes client id, secret, username and password on
`{Scheme}CredentialsBuilder::init(...)` and acquires its token automatically, exactly as client
credentials does. **Authorization code** takes client id, secret and redirect URI, and is not automatic:
`buildAuthorizationUrl(?string $state, ?array $additionalParams)`, then
`fetchToken(string $code, ?array $additionalParams)`, then rebuild the client with that token. It
generates **no `oAuthTokenProvider` / `oAuthOnTokenUpdate`**, and without a token a call throws
`\InvalidArgumentException` rather than making a request.

Where an API requires more than one scheme, set every credentials builder the operation needs; the
client composes them, and `doc/controllers/*.md` says which operation needs which.

To see what this SDK accepts, read the `->…Credentials(…)` setters on `src/{Client}Builder.php`, then
`init(...)`'s signature in `src/Authentication/` for the required parameters. The accessor is
`get{Scheme}Credentials()`, or bare `get{Scheme}()` under the single-scheme OAuth asymmetry above, so
read `src/ConfigurationInterface.php` rather than grepping only for the suffixed name.

## Notes

- **There is no `fromEnvironment` factory.** Nothing in `src/` reads environment variables. Read them
  yourself with `getenv()` / `$_ENV` (or a `.env` loader) and pass the values in — never hardcode a
  secret, and never commit one.
- A generated SDK may also carry **deprecated flat setters** for individual auth parameters (e.g.
  `->oAuthClientId('…')`) when the API has exactly one scheme. Their docblocks point at the credentials
  setter; use the credentials builder instead.
- Setting credentials is a **construction-time** decision: there is no setter on a built client. See
  **php-client-initialization** for `toBuilder()` / `withConfiguration()`.

## Next

- Step 3, make your first call → **php-calling-endpoints**
- A call that comes back `401` or `403` → **php-error-handling**
