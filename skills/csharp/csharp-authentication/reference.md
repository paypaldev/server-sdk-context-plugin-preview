# Authentication reference (APIMatic C#)

The full matrix of auth schemes the APIMatic C# generator emits. Every scheme lands in
`PaypalServerSdk.Standard/Authentication/` as one `{Scheme}Manager.cs`, and the *shapes* below are identical
across SDKs — only the **names** are generated, from the security-scheme names in the API spec. Confirm
each one against `PaypalServerSdkClient.cs` and `doc/auth/`.

## Naming pattern

`{Scheme}` is the emitted scheme name in PascalCase. Most pieces derive from it — but **the derivation
is not reliable enough to write code from, and it is not always the spec's own scheme name.** Read the
client `Builder`'s method list and the `Authentication/` folder, and take these as the shapes to
recognise rather than names to construct:

| Piece | Usual shape | Where it lives |
| --- | --- | --- |
| Credential model | `sealed class {Scheme}Model` + nested `Builder` | inside `{Scheme}Manager.cs` — there is no `{Scheme}Model.cs` |
| Manager | `{Scheme}Manager : AuthManager, I{Scheme}Credentials` — accessibility varies: `internal` in many builds, but **`public`** where a callback exposes it (an OAuth `OAuthTokenProvider` is `Func<{Scheme}Manager, …>`, so the manager must be reachable) | `{Scheme}Manager.cs` |
| Read interface | `interface I{Scheme}Credentials` — **or bare `I{Scheme}`** | that interface's own file |
| Client `Builder` setter | `.{Scheme}Credentials({Scheme}Model)` — **or bare `.{Scheme}(...)`** | `PaypalServerSdkClient.cs` |
| Client getters | the manager getter mirrors the setter's name — `client.{Scheme}Credentials` when the setter carries the suffix, bare `client.{Scheme}` when it does not; the model getter is always `client.{Scheme}Model` | `PaypalServerSdkClient.cs` |
| Config bind target | `class {Scheme}ModelOptions`, under the JSON key matching the setter | `{Scheme}Manager.cs` |

**Grep the client `Builder` for the setter, `Authentication/` for the interface file name, and the
client class for the manager getter** — none of the three is inferable, and `{Scheme}ModelOptions` can
differ in casing from both the model and the manager. The getter always mirrors the setter, so if the
setter dropped the suffix the getter has too.

## The non-OAuth schemes

Each is one `.{Scheme}Credentials(new {Scheme}Model.Builder(…).Build())` call — the SKILL covers using
them. What the manager then does with the values, which never appears in your code:

| Scheme | What the manager registers, in its constructor |
| --- | --- |
| Basic | `Authorization: Basic <base64(user:pass)>`, encoded with **`Encoding.ASCII`** |
| API key — header | `.Header(header => header.Setup("<name>", value).Required())`, once per parameter |
| API key — query | `.Query(query => query.Setup("<name>", value).Required())`, once per parameter |
| Bearer / access token | `Authorization: Bearer <token>` — set once, **never refreshed** |

A scheme may carry more than one parameter; the `Builder` constructor's arity is the answer.

## OAuth 2.0 — client credentials (the SDK drives it)

The only grant the SDK completes for you: given a client id and secret the manager acquires a token on
the first call that needs the scheme, caches it, and re-acquires when it expires. The construction
sample is in the SKILL; these are the setters it can carry.

| Optional setter | Type | Purpose |
| --- | --- | --- |
| `OAuthToken(Models.OAuthToken)` | the generated token model | seed a token you already hold |
| `OAuthScopes(List<{ScopesEnum}>)` | generated `[EnumMember]` enum | only when the spec declares scopes |
| `OAuthClockSkew(TimeSpan?)` | `TimeSpan` — **not** a number of seconds | treat a token as expired this early |
| `OAuthTokenProvider(Func<{Scheme}Manager, OAuthToken, Task<OAuthToken>>)` | callback | load a stored token instead of fetching |
| `OAuthOnTokenUpdate(Action<OAuthToken>)` | callback | fires on every refresh — persist here |

Lifecycle, in order: nothing happens at `Build()`; the first call on an operation using the scheme starts
everything; the manager reuses the cached token while it is unexpired under `OAuthClockSkew`, otherwise
calls your `OAuthTokenProvider` if set and its own `FetchTokenAsync()` if not; `OAuthOnTokenUpdate` fires
after each refresh; then `Authorization: Bearer <AccessToken>` goes on the request.

**A `null` `Expiry` counts as expired.** A seed token without one is replaced on the first call, and one
your provider returns without one raises `ApiException`. Persist the expiry — `Models.OAuthToken`'s
public constructor is `(accessToken, tokenType, expiresIn, scope, expiry, refreshToken)`, everything
after `tokenType` optional, so pass `expiry:` (a UTC Unix timestamp in seconds) by name.

`client.{Scheme}Credentials.IsTokenExpired()` is a **different** check: it reads only the seed token
passed to `.OAuthToken(...)`, never the one the manager fetched and is sending, ignores
`OAuthClockSkew`, reports a `null` `Expiry` as *not* expired, and throws
`InvalidOperationException("OAuth token is missing.")` when there was no seed token.

## OAuth 2.0 — authorization code (3-legged)

**Not automatic.** The manager builds the authorization URL and exchanges the code; you drive the flow
and then rebuild the client with the token. The `Builder` constructor takes the client id, the client
secret and the redirect URI.

```csharp
// 1. send the user here — BuildAuthorizationUrl is async, so it must be awaited
string url = await client.{Scheme}Credentials.BuildAuthorizationUrl(state: "...");

// 2. your redirect endpoint receives ?code=...
Models.OAuthToken token = await client.{Scheme}Credentials.FetchTokenAsync(authorizationCode);

// 3. re-attach it — there is no setter on a built client
client = client.ToBuilder()
    .{Scheme}Credentials(client.{Scheme}Model.ToBuilder().OAuthToken(token).Build())
    .Build();
```

Also on the interface: `BuildAuthorizationUrl(string state, Dictionary<string, object>
additionalParameters)`, `FetchToken`/`FetchTokenAsync(string authorizationCode, Dictionary<string,
object> additionalParameters)`, `RefreshToken`/`RefreshTokenAsync(...)` and `IsTokenExpired()`. Skipping
step 3 leaves the scheme unsatisfied and the first call fails on validation.

## OAuth 2.0 — resource owner password, and implicit

**Resource owner password** is the authorization-code shape minus the browser step: no
`BuildAuthorizationUrl`, and the `Builder` constructor takes the username and password (plus the client
id and secret in the variant that uses them). Call `FetchTokenAsync()` and re-attach as in step 3 above.

**Implicit** is authorization-URL only: `BuildAuthorizationUrl(...)` and `IsTokenExpired()` and **no**
`FetchToken` — the token arrives in the redirect fragment and you attach it with `.OAuthToken(...)`
through `ToBuilder()`.

## Custom auth

**The scheme's parameter list decides how much surface you get — but not whether it works.** Read
`Authentication/{Scheme}Manager.cs`.

- **It declares no parameters** → the manager is a bare
  `// TODO: Add your custom authentication here` stub with nothing to store and nothing to set.
- **It declares parameters** → you get the ordinary apparatus around them: the manager takes them as
  constructor arguments, and a `{Scheme}Model`, `{Scheme}ModelOptions`, `I{Scheme}Credentials` and a
  `.{Scheme}Credentials(model)` setter are generated just as for a built-in scheme.

**In both shapes nothing reaches the wire.** The parameterised manager only *assigns its arguments to
properties*; the `Parameters(parameters => parameters.Header(...))` call that would apply them is emitted
**commented out**, beside the same `// TODO`. Some builds also carry an `internal`
`AuthUtility.AppendCustomAuthParams(config, request)` — likewise a stub. So a custom scheme that looks
fully configurable still authenticates nothing: values you set are stored and silently dropped.

Treat a custom scheme as **"this SDK cannot authenticate this scheme yet"** — it works once the SDK
describes it properly, or send the credential yourself through a `DelegatingHandler` on the
`HttpClientInstance` seam (**csharp-testing** shows the seam). Never edit the SDK. And do not read
`internal` as the tell: managers are `internal` in most builds regardless of scheme, so reach every
scheme through its public `I{Scheme}Credentials` interface.

## Combined / multiple schemes

There is no combined credentials object: set **every** scheme the operations you call require, and the
operation composes them. Each operation states its own requirement in its request builder —
`.WithAuth("<scheme>")` for one, `.WithOrAuth(...)` where any one listed suffices, `.WithAndAuth(...)`
where all are applied — and an operation carrying none of the three needs no credentials at all. Two
operations in the same controller can differ; the **Authentication** section of
`doc/controllers/{group}.md` restates it.

One asymmetry worth knowing: a client-credentials manager registers its header only inside `Apply()`, so
it **always passes** the up-front validation. With no client id or secret set, the call still reaches
the token endpoint and the failure surfaces from *that* request rather than as a missing-credential
error.

**No auth** is the degenerate case: an API — or an individual operation — may need no credentials at
all. Build the client without any credential setter; nothing validates.

## Binding credentials from configuration

`PaypalServerSdkClient.FromConfiguration(configuration.GetSection("PaypalServerSdk"))` reads one
`{Scheme}Credentials` object per scheme into the matching `{Scheme}ModelOptions`, then rebuilds the model
through `{Scheme}Model.FromOptions(...)`. `{Scheme}ModelOptions` carries the credential values and
`OAuthClockSkew` only — `OAuthTokenProvider` and `OAuthOnTokenUpdate` are delegates with no JSON form and
can be set in code alone.

> **`FromOptions` is `internal` — that route is the SDK's, not yours.** Binding a `{Scheme}ModelOptions`
> yourself and calling `{Scheme}Model.FromOptions(options)` is **`CS0117`**; the options class is public
> with public setters, which makes it look usable, but the only way in is
> `PaypalServerSdkClient.FromConfiguration(...)`. The same holds for `HttpClientConfiguration.FromOptions` and
> `ProxyConfigurationBuilder.FromOptions`. The JSON key for each scheme is the **setter name** from the
> **Authentication** table, and the inner keys are the PascalCase `Builder` argument names.

## Discovering what a specific SDK uses

1. `PaypalServerSdkClient.cs` — list the `Builder` methods that take a `{Scheme}Model`. Most end in `Credentials`, but a single OAuth 2 scheme drops that suffix, so match on the model parameter type rather than the method name. This is the **source of
   truth** for what the SDK accepts; nothing else is.
2. `Authentication/{Scheme}Manager.cs` — the `Builder` constructor for the required credentials, its
   fluent setters for the optional ones, and whether the manager overrides `Apply` (self-refreshing) or
   exposes `FetchToken`/`BuildAuthorizationUrl` (you drive it).
3. `doc/auth/*.md` — the same parameters with a worked snippet, and `doc/controllers/{group}.md` for
   which operations require which scheme.
