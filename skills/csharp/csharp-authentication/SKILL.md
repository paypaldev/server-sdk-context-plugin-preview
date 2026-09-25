---
name: 'csharp-authentication'
description: 'Set credentials on the PayPal Server SDK C# SDK. Load before configuring any scheme, or when a call comes back 401 or 403. The setter name won''t tell you it can drop or double the `Credentials` suffix, that required values are positional on the model''s constructor and throw `ArgumentNullException` there rather than on the first call, or which getter reads the model back.'
---

# Authenticating an APIMatic C# SDK client

The API spec decides which schemes exist. Set every scheme your endpoints need **while you build the
client** — a built client has no credential setter (see **csharp-client-initialization**).

## Finding which schemes this SDK accepts

1. `PaypalServerSdkClient.cs` — the credential setters on `PaypalServerSdkClient.Builder` are **the source of
   truth**.
2. `Authentication/` — one `{Scheme}Manager.cs` per scheme, holding the manager and the
   `sealed class {Scheme}Model`, plus an `I{Scheme}Credentials.cs`. A **custom** scheme that declares no
   parameters has no model and no client setter at all — see [reference.md](reference.md).
3. `doc/auth/*.md` — a page per scheme listing each parameter with its type, setter and getter.

**Never derive the setter name — take it from the Authentication table in csharp-getting-started, which
resolves it.** The usual form is the emitted scheme name with `Credentials` appended
(`.BasicAuthCredentials(...)`), and a grant-named scheme sometimes doubles the word
(`.Oauth2ClientCredentialsCredentials(...)`). The suffix is dropped **only** when the SDK has exactly one
scheme *and* that scheme is an OAuth 2 grant type (`.ClientCredentialsAuth(...)`, interface
`IClientCredentialsAuth`). Ending in `Auth` is **not** the trigger — a `BasicAuth` scheme alongside others
still emits `.BasicAuthCredentials(...)` and `IBasicAuthCredentials`. Read the `Builder`'s method list in
`PaypalServerSdkClient.cs` and the file names under `Authentication/`; the shapes to recognise are in
[reference.md](reference.md).

> **This SDK's schemes, setters, credential models and required arguments are already resolved** in the
> *This SDK's map* table at the top of **csharp-getting-started**. Read that first — everything below is
> the shape and the traps, not the names.

## Which schemes an operation needs

Grep the operation's request builder for `WithAuth` / `WithOrAuth` / `WithAndAuth`.

> **The string inside is the *spec's* security-scheme name, not the emitted C# name.** It is a key into
> the `AuthManagers` dictionary built in `PaypalServerSdkClient.cs`, so it may have no setter, no
> `{Scheme}Model`, no `Authentication/` file and no `doc/auth/` page under that spelling — a spec scheme
> called `BearerAuth` can be wired to a manager the SDK exposes as `ClientCredentialsAuth`. Resolve the
> key through that `.AuthManagers(...)` dictionary to get the scheme in the **Authentication** table
> above; do not match it against setter names. On a single-scheme SDK the mismatch is harmless, but with
> two or more it is the difference between configuring the right credential and the wrong one.

- **no match anywhere in the controller folder** — no operation requires a scheme. A credential-less
  client works, and anything you configure never reaches the wire.
- **`WithAndAuth(a, b)`** — configure both, or the call throws `AuthValidationException` while the
  request is built.
- **`WithOrAuth(...)`** — any one alternative satisfies it. The SDK applies the **first *satisfiable*
  alternative in listed order** and sends only that one: an earlier alternative you left unconfigured is
  skipped, and a later one you did configure is used. So if the choice matters (different rate limits,
  different audit identity), configure exactly the one you intend, and read the argument order to see which
  wins when you configure more than one.

  > **An OR group can mix the two failure kinds, and then the OAuth one governs.** Where the group lists
  > both a parameter-registering scheme and an OAuth grant, a client with **no** credentials does not throw
  > `AuthValidationException` — the OAuth alternative passes up-front validation, so what you get is a token
  > request through your handler and then an `ApiException` (see the callout below). Reading the group as
  > "two header schemes are listed, so it must fail before the wire" is wrong. Check whether any alternative
  > in the group is a grant before predicting the failure.
- **`.AddAndGroup(g => g.Add("a").Add("b"))` nested inside a `WithOrAuth`** — an AND requirement that does
  **not** appear as `WithAndAuth`. Some SDKs express every AND this way, so grepping only for `WithAndAuth`
  concludes "no operation needs two schemes at once" when several do. Read the whole auth expression, not
  just its opening call.

## The shape: build a model, hand it to the client builder

```csharp
using PaypalServerSdk.Standard;
using PaypalServerSdk.Standard.Authentication;

var client = new PaypalServerSdkClient.Builder()
    .{Scheme}Credentials(
        new {Scheme}Model.Builder(
                System.Environment.GetEnvironmentVariable("API_CLIENT_ID"),      // required: positional
                System.Environment.GetEnvironmentVariable("API_CLIENT_SECRET"))  // null => ArgumentNullException
            .{OptionalParam}(...)                                                // optional: fluent setter
            .Build())
    .Build();
```

- **Required credentials are positional on the `Builder`'s constructor**, each assigned
  `?? throw new ArgumentNullException(...)` — an unset environment variable fails there, not on the
  first call. Every parameter also gets a same-named fluent setter; the optional ones are the only
  ones **not** null-checked.
- `{Scheme}Model` is `sealed` with an `internal` constructor and `internal` properties: the `Builder`
  is the only way to make one; `model.ToBuilder()` derives a variant.
- Write `System.Environment` in full — `using PaypalServerSdk.Standard;` also brings the SDK's own
  `Environment` enum into scope, so with `System` imported a bare `Environment` is ambiguous.

> **`Build()` drops an unconfigured scheme silently.** The client `Builder` seeds every scheme with an
> empty model, then replaces any whose required values are still `null` with `null` — no exception, no
> log. `client.{Scheme}Model == null` is the in-SDK signal — assert on it at startup.
>
> **How the first call then fails depends on the scheme kind, and the two are not interchangeable:**
>
> - **Schemes that register a required wire parameter** — custom header, custom query, basic, static
>   bearer — fail *before the request leaves*, throwing
>   `APIMatic.Core.Types.Sdk.Exceptions.AuthValidationException` with a message naming each missing
>   parameter. It derives from `ArgumentNullException`, **not** `ApiException`, so an `ApiException` catch
>   never sees it, and no HTTP request is made — a stub handler records **zero** calls.
> - **OAuth 2 grant schemes register nothing outside their `Apply()`**, so the up-front validation finds
>   nothing missing and **always passes** — there is no `AuthValidationException` on this path at all. What
>   happens instead depends entirely on **how the token endpoint answers**, and a real `POST` to it has
>   already travelled through your handler by then. Three outcomes, all reachable with empty credentials:
>   - **The token call succeeds** — the SDK caches the token and your call goes out authenticated. A
>     credential-less client therefore **succeeds** against a stub that answers the token request, which is
>     what a stubbed test normally does. No exception, and two recorded requests.
>   - **It returns 2xx but the token has no usable expiry** — `PaypalServerSdk.Standard.Exceptions.ApiException`,
>     message `OAuth token is expired. A valid token is needed to make API calls.`, `HttpContext == null`,
>     `ResponseCode == -1`, one recorded request.
>   - **It returns an error** (what a real server does with empty credentials) —
>     `OAuthProviderException`, a typed subclass of `ApiException` carrying the provider's own
>     `ResponseCode` and a real `HttpContext`.
>
>   All three are caught by `catch (ApiException)`. So do **not** write a test asserting "a client with no
>   credentials throws": against a stub that answers the token request it does not. Assert on the outcome
>   you actually stubbed.
>
> Getting this backwards is the most common way a stubbed test goes wrong: against an OAuth scheme a
> credential-less client produces one unexplained extra request and an exception type you did not expect.
> Check which kind you have — `{Scheme}Manager` overriding `Apply` with a `FetchToken`/`IsTokenExpired`
> pair is a grant; one that sets its header in the constructor is not.

## Basic auth

```csharp
.{Scheme}Credentials(new {Scheme}Model.Builder(username, password).Build())
```

Sends `Authorization: Basic <base64(username:password)>`, encoded with `Encoding.ASCII` — non-ASCII
credentials are **mangled rather than rejected**.

## API key — header or query parameter

```csharp
.{Scheme}Credentials(new {Scheme}Model.Builder(apiKey).Build())
```

Placement and wire name are fixed in the manager's constructor and never appear in your code. A scheme
can take **more than one** parameter — the `Builder` constructor's arity is the answer.

## Bearer / access token

**Check the manager before assuming this is a static token.** A scheme named `BearerToken` or
`BearerAuth` may be either:

- **A static token** — one required `Builder` parameter, the token itself. The manager sets
  `Authorization: Bearer <token>` in its constructor and does nothing else: it never fetches or
  refreshes, so build a new client when the token changes.
- **An OAuth client-credentials grant wearing a bearer name** — the `Builder` then takes **two**
  parameters (`oAuthClientId`, `oAuthClientSecret`), and the manager overrides `Apply` and exposes
  `FetchToken`/`FetchTokenAsync`/`IsTokenExpired`. It acquires and refreshes on its own; treat it as
  the client-credentials section below.

**The `Builder` constructor's arity tells you which** — one argument or two. Passing one to a
two-argument constructor is `CS7036`.

## OAuth 2.0 — client credentials

**This manager drives the exchange itself**: given a client id and secret it acquires a token on the
first call that needs the scheme, caches it, and re-acquires when that one expires. Other grants leave
it to you, exposing `FetchToken`/`RefreshToken` or a `BuildAuthorizationUrl` you drive yourself — **the
manager that overrides `Apply` is the one that refreshes on its own**, so read the manager file.

```csharp
.{Scheme}Credentials(
    new {Scheme}Model.Builder(
            System.Environment.GetEnvironmentVariable("API_OAUTH_CLIENT_ID"),
            System.Environment.GetEnvironmentVariable("API_OAUTH_CLIENT_SECRET"))
        .OAuthClockSkew(TimeSpan.FromSeconds(30))       // a TimeSpan, not a number of seconds
        .OAuthOnTokenUpdate(token => SaveToken(token))  // fires on every refresh — persist here
        .OAuthTokenProvider(async (manager, current) => // supply a stored token instead of fetching
            LoadToken() ?? await manager.FetchTokenAsync())
        .Build())
```

Those three setters plus `OAuthToken` are emitted on every client-credentials model; the credential
parameters come from the spec, so confirm the `Builder`'s names in the manager file. **`OAuthScopes` is
not among them unless the spec declared scopes on this grant** — usually only the authorization-code
one does. Where it exists it takes `List<{ScopesEnum}>`, a generated `[EnumMember]`-mapped enum in
`PaypalServerSdk.Standard.Models` — **pass its members, never raw strings**. The scopes themselves are the
`Scopes` table in that grant's `doc/auth/*.md` page; `doc/controllers/{group}.md` names the scheme an
operation requires but never its scopes.

Two traps, with the full lifecycle, in [reference.md](reference.md): a **`null` `Expiry` counts as
expired**, so persist the expiry or every restart re-fetches; and
`client.{Scheme}Credentials.IsTokenExpired()` reads **only the seed token passed to `.OAuthToken(...)`**,
never the one the manager fetched and is sending, so it is not the health check it looks like.

## Reading credentials back off the client

| Property | Type | What it gives you |
| --- | --- | --- |
| `client.{Scheme}Credentials` — **or bare `client.{Scheme}`, matching whatever the setter is called** | `I{Scheme}Credentials` or bare `I{Scheme}` | the auth manager behind its interface — **never `null`**, configured or not |
| `client.{Scheme}Model` | `{Scheme}Model` | the model you supplied — `null` if `Build()` discarded it |

`I{Scheme}Credentials` exposes a getter for each **credential value** plus an `Equals(...)` overload; a
client-credentials grant adds `FetchToken`, `FetchTokenAsync` and `IsTokenExpired`.

> **Some values you set through a fluent setter on the model's `Builder` are not readable back through
> either route** — check the interface rather than assuming either way. Credential-shaped values usually
> *are* there (an `OAuthToken` or `OAuthScopes` you supplied typically appears on the interface);
> the tuning knobs and callbacks are not. `OAuthClockSkew` is the one that catches people: it is a
> `TimeSpan?` value, not a callback,
> and it is absent from `I{Scheme}Credentials` on every SDK — `credentials.OAuthClockSkew` is
> **`CS1061`**. The `client.{Scheme}Model` route fails identically, because the model's properties are
> `internal`: that getter is a null-check, never a readable value. The only route is a downcast to the
> concrete manager, `(({Scheme}Manager)client.{Scheme}Credentials).OAuthClockSkew` — and that compiles
> only when the manager class is `public`, which varies. The same applies to the token-provider and
> on-token-update callbacks. **If you need to assert your own configuration at startup, keep the value you
> passed in rather than trying to read it back.**

## Combining schemes — the operation decides, not the client

There is no combined credentials object: set **every** scheme the operations you call require, and each
operation composes what it needs. **Two operations in the same controller can differ**, and one may
need no credentials at all — read the operation in `PaypalServerSdk.Standard.Controllers`, or the
**Authentication** section of `doc/controllers/{group}.md`.

## More schemes

For OAuth 2 **authorization code (3-legged)**, **resource-owner password**, **implicit**, **custom
auth** and **no-auth**, the AND/OR composition rules, and binding credentials from configuration, see
[reference.md](reference.md).

## Notes

- **Rotating credentials means a new client.** `client.ToBuilder()` carries the credential models over,
  so `client.ToBuilder().{Scheme}Credentials(newModel).Build()` is the rotation — but it does **not**
  carry the HTTP client configuration; see **csharp-client-initialization**.
- `Equals(...)` on a credentials interface is a **credential-comparison overload**, not
  `object.Equals`; it dereferences its argument, so `null` throws.
- **A missing credential is not an `ApiException` — unless the scheme is an OAuth grant.** For
  parameter-registering schemes it is `AuthValidationException`, an `ArgumentNullException`, raised before
  the request leaves; for OAuth grants it is an `ApiException` raised from the token request. See the
  callout under **The shape** above, and **csharp-error-handling** for where each failure class lands.

## Next

- Step 3, make your first call → **csharp-calling-endpoints**
- A call that comes back `401` or `403` → **csharp-error-handling**
