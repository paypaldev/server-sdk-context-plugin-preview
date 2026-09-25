---
name: 'csharp-client-initialization'
description: 'Construct and configure the PayPal Server SDK C# SDK client. Load before you call the builder or wire the client into an application. The method list won''t tell you the client is `sealed` with a nested `Builder` and no public constructor, that controllers are get-only properties rather than types you construct, or that one client is built once and reused.'
---

# Initializing an APIMatic-generated C# SDK client

Names carrying a token are concrete for this SDK; replace the `{...}` placeholders with the real names
from its source:

- `{Controller}` — a controller class in `PaypalServerSdk.Standard.Controllers`, and the client property for it.
- `{MEMBER}` — a member of the `Environment` enum; `{SERVER}` — a member of the `Server` enum.
- one credential model and setter per auth scheme — **the exact names are in the SDK map at the top of
  csharp-getting-started**; do not derive them, the suffix is dropped for some schemes.
- `{ConfigVar}` — a builder setter generated from a configuration variable declared in the API spec.

## The shape: a nested Builder, no public constructor

`PaypalServerSdkClient` is a `public sealed class` and its constructor is **private**. Every instance comes
out of the nested `Builder` — directly, from the static `FromConfiguration(IConfigurationSection)`, or
from `ToBuilder()` on a built client.

```csharp
using System;
using PaypalServerSdk.Standard;
using Environment = PaypalServerSdk.Standard.Environment;  // else CS0104: ambiguous with System.Environment

PaypalServerSdkClient client = new PaypalServerSdkClient.Builder()
    .Environment(Environment.{MEMBER})
    .HttpClientConfig(config => config
        .Timeout(TimeSpan.FromSeconds(30)))          // TimeSpan — there is no seconds-as-int overload
    // credentials — see csharp-authentication
    .Build();
```

The client implements the SDK's own **read-only** `PaypalServerSdk.Standard.IConfiguration` — a getter per
configuration variable, the credential getters and `GetBaseUri(Server alias = Server.{SERVER})` — not
`Microsoft.Extensions.Configuration.IConfiguration`. An SDK generated with interfaces declares
`: IPaypalServerSdkClient` instead, and that interface extends `IConfiguration`.

The `Builder` methods, in `PaypalServerSdkClient.cs` — **read that class for this SDK's real set**:

| Method | Argument | Notes |
| --- | --- | --- |
| `Environment(Environment)` | enum member | always generated |
| `{ConfigVar}(...)` | one per spec configuration variable | server template variables and global parameters; **not** timeout or retries |
| the credential setter named in the SDK map | a **built** model | one per scheme; throws `ArgumentNullException` on null |
| `HttpClientConfig(Action<HttpClientConfiguration.Builder>)` | an action | timeout, retries, proxy, your `HttpClient` |
| `HttpCallback(HttpCallback)` | an `HttpCallback` | observation only |
| `Build()` / `static FromConfiguration(IConfigurationSection)` | — / a section | build; or a pre-populated `Builder` |

This SDK has built-in logging, so the `Builder` also carries
`LoggingConfig()` and `LoggingConfig(Action<LogBuilder>)` — see **csharp-configuration-resilience**.

> **`doc/client.md` advertises builder methods that do not exist.** Its *Builder Class* table lists
> `Timeout(TimeSpan)`, `HttpClientConfiguration(Action<...>)` and
> `{Scheme}Credentials(Action<{Scheme}Model.Builder>)` — the code has none of the three. The `.cs` file
> is authoritative for method existence; the doc table is not.

`Build()` **never rejects an incomplete credential model** — one missing a required value is silently
set to `null` and the client builds anyway, failing only at call time (**csharp-authentication**).

`HttpClientConfig` is an **action, not an object**: it hands you the `HttpClientConfiguration.Builder`
the client builder already holds, so two `.HttpClientConfig(...)` calls **accumulate**. Mind the
asymmetry — the builder method is `HttpClientConfig`, the property on the built client is
`HttpClientConfiguration`. **Do not assume a retry count, a timeout or a retried-status-code list** —
see **csharp-configuration-resilience**.

## Choosing the environment / base URL

Environments are members of an `enum Environment` in `PaypalServerSdk.Standard` (`Environment.cs`). **Read the
enum for the real member names before naming one** — there
may be no `Production` at all. Match each member to the URL it resolves to in the environments map at
the top of `PaypalServerSdkClient.cs`.

```csharp
var client = new PaypalServerSdkClient.Builder()
    .Environment(Environment.{MEMBER})
    .{ConfigVar}("...")                             // only where this SDK declares configuration variables
    .Build();

string url = client.GetBaseUri();                   // default server of the selected environment
string alt = client.GetBaseUri(Server.{SERVER});
```

**There is no base-URL override** — `GetBaseUri` is a getter only, and your only levers are
`Environment(...)` and the spec's configuration variables. See **csharp-configuration-resilience** for
what that rules out and how to reach a host no `Environment` member covers.

> **Omit `.Environment(...)` and you still get one.** The `Builder` initialises the field to a default
> chosen by the API definition, not by you, and **that default may be the live environment** — nothing
> warns, logs or fails. Read the initialiser on the `Builder`'s `environment` field in
> `PaypalServerSdkClient.cs` to see which member your SDK actually starts from, and set it explicitly
> whichever it turns out to be.
>
> **A second, separate trap:** `Environment` is emitted with no explicit values, so its members number
> from zero in declaration order. `default(Environment)`, `(Environment)0`, a zero-initialised field and
> a failed `Enum.TryParse` therefore all mean **the first member in `Environment.cs`** — which is not
> necessarily the `Builder`'s default. The two coincide only when the API definition happens to list its
> default first. Never let an environment reach the builder as an unchecked `default`.

## Configuration from a bound `IConfiguration` section

```csharp
var section = configuration.GetSection("PaypalServerSdk");     // an IConfiguration you already have

var client = PaypalServerSdkClient.FromConfiguration(section);         // bound, then built
var other = PaypalServerSdkClient.Builder.FromConfiguration(section)   // bound, then overridden in code
    .Environment(Environment.{MEMBER}).Build();
```

`"PaypalServerSdk"` is the section key the generated documentation uses. The bind target is the
nested `PaypalServerSdkClientOptions` class at the bottom of `PaypalServerSdkClient.cs`, and **its property names
are exactly the JSON keys**; nested keys come from `HttpClientConfigurationOptions`,
`{Scheme}ModelOptions` and `LoggingConfigOptions`. Read those, not the generated sample JSON — it has
at least one key that binds to nothing. **The `Environment` key takes the C# enum member name, not the
`[EnumMember]` wire value** — write `"NonProductionServer"`, not `"Non-Production server"`. `ConfigurationBinder`
matches enum members **case-insensitively**, so a wire value that differs from the member name only in case
(`"production"` for `Production`) does bind, and several SDKs' own sample JSON relies on that — do not
"correct" it. A wire value differing by more than case (spaces, hyphens) throws `InvalidOperationException`.
> **One configuration failure is loud, and it is the likely one.** A credential block that is *present but
> incomplete* — say `BasicAuthCredentials` with a `Username` and no `Password`, or carrying only keys the
> options class does not declare — throws
> `ArgumentNullException` naming the missing argument, straight out of `FromConfiguration`, because binding
> reaches `{Scheme}Model.FromOptions` and that calls the model `Builder`'s required-argument constructor.
> This is the opposite of the code route, where `Build()` discards an incomplete model in silence. So:
> **omit a credential block entirely and you get a null model; half-fill one and you get an exception at
> startup.** Catch it where you build the client, and do not assume configuration binding fails quietly.
>
> The one exception is an **empty** block (`"{Scheme}Credentials": {}`): the JSON provider emits no keys for
> it, so the options object stays null and you get the *silent* outcome, indistinguishable from omitting it.
> A placeholder block you meant to fill in later therefore fails at the first call, not at startup.

**A missing or wrongly-named section is silent** (configuration keys are matched
case-insensitively, so only a genuinely different name misses — a case difference binds fine): the bind yields
`null` and you get an untouched default builder — no credentials, default environment, no error.

The SDK references the configuration **binder** only — `Microsoft.Extensions.Configuration.Binder`, which
brings just `…Configuration.Abstractions`. You must add packages yourself, and the first one you need is
not a provider: **`Microsoft.Extensions.Configuration`** for `ConfigurationBuilder` itself, or you get
`CS0234` before ever reaching a provider. Then `…Configuration.Json` for `AddJsonFile` and
`…Configuration.EnvironmentVariables` for `AddEnvironmentVariables`. `CreateFromEnvironment()` is
**`internal`**, so no consumer assembly can call it — read the variables yourself.

## Accessing controllers — get-only properties on the client

You do **not** construct controllers; their constructors are `internal`. Each is a get-only property on
the client, backed by a `Lazy<T>` field, so it is created on first access and shared thereafter:

```csharp
var controller = client.{Controller};
```

The property name is the controller class name — the endpoint group in PascalCase, plus a postfix
**only when the SDK was generated with the controller-postfix option on**, and dropped again when the
group name already ends with it. **Read the names off `PaypalServerSdkClient.cs`, or the *Controllers* table
in `doc/client.md`.** The classes live in `PaypalServerSdk.Standard.Controllers`; with generated interfaces the
declared type is `I{Controller}` while the property name is unchanged. Calling one:
**csharp-calling-endpoints**.

## Client lifetime and reuse

The constructor builds the auth managers, the global configuration and the controller holders **once**,
and every configuration member is get-only. Treat the client as **immutable and long-lived**: build it
at startup and reuse it — a client per request throws away connection reuse and any OAuth token the
auth manager has refreshed. It is not `IDisposable`; there is no shutdown or close call.

To vary a built client, round-trip through the builder — also how a refreshed token gets attached (see
**csharp-authentication**):

```csharp
var other = client.ToBuilder()
    .HttpClientConfig(config => config.Timeout(TimeSpan.FromSeconds(120)))  // re-apply: see below
    .Build();
```

> **`ToBuilder()` does not carry the HTTP client configuration.** It copies the configuration variables,
> callback, logging configuration and credential models, but starts a **fresh**
> `HttpClientConfiguration.Builder` — timeout, retries and any injected `HttpClient` revert to
> defaults.

## Dependency injection

The client is immutable once built, so register it as a **singleton**; the SDK generates no DI extension
method. Qualify or alias `IConfiguration` in any file importing both `PaypalServerSdk.Standard` and
`Microsoft.Extensions.Configuration` — each declares one. `PaypalServerSdkClient` is `sealed` and implements nothing covering its
operations, so inject a narrow interface of your own — that wrapper is also the seam your tests need
(see **csharp-testing**).


## Next

- Configure authentication → **csharp-authentication**
- Make your first call → **csharp-calling-endpoints**
- Tune retries/timeouts/proxy/logging → **csharp-configuration-resilience**