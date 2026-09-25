---
name: 'java-client-initialization'
description: 'Construct and configure the PayPal Server SDK Java SDK client. Load before you call the builder or wire the client into an application. The method list won''t tell you `httpClientConfig` takes a lambda rather than an object, that controllers are accessors on the client, or that a cloned client keeps some settings and drops others.'
---

# Initializing an APIMatic-generated Java SDK client

Replace the `{...}` placeholders below with the real names from the SDK source:

- `{Api}Client` — the single client class in the SDK's root package.
- `{Controller}` — a controller class; its package and class suffix are both named per SDK, so take both
  from the `import` block of `{Api}Client.java`.

## The shape: a nested Builder, no public constructor

The client is `public final class {Api}Client` and its constructor is **private**. The only way to build
one is through the nested `Builder`:
```java
import {rootPackage}.{Api}Client;
import {rootPackage}.Environment;

{Api}Client client = new {Api}Client.Builder()
        .environment(Environment.{MEMBER})
        .httpClientConfig(configBuilder -> configBuilder
                .timeout(30))
        // this API declares a scheme, so a credential model belongs here — see java-authentication
        .build();
```

This SDK ships no generated interfaces, so the class is declared `implements Configuration` and no
client interface is generated — the concrete class is the only type you can inject.

`Configuration` is a **read-only interface** the client implements — `getEnvironment()`,
`getHttpClientConfig()`, `getBaseUri()`, `getBaseUri(Server)`. It is *not* something you construct and
pass in. Everything configurable is a `Builder` method; open `{Api}Client.java` and read the `Builder`
class for the exact set, which in this build includes a `loggingConfig` setter.

The `Builder` methods that are always generated:

| Method | Purpose |
| --- | --- |
| `environment(Environment environment)` | selects the environment (and therefore the base URL) |
| `httpClientConfig(Consumer<HttpClientConfiguration.Builder> action)` | transport tuning — timeouts, retries, proxy, a custom OkHttp client |
| `httpCallback(HttpCallback httpCallback)` | a before-request / after-response hook — see **java-configuration-resilience** and **java-testing** |
| `build()` | produces the client |

There is also a **deprecated** `timeout(long)` directly on the client `Builder`. Prefer
`httpClientConfig(b -> b.timeout(...))`; the top-level one only forwards to it.

## `httpClientConfig` takes a lambda, not an object

This is the shape that catches people out. You do **not** build an `HttpClientConfiguration` and pass it
in — you pass a `Consumer` that receives the builder and mutates it:

```java
{Api}Client client = new {Api}Client.Builder()
        .httpClientConfig(configBuilder -> configBuilder
                .timeout(30)                 // SECONDS, not milliseconds
                .numberOfRetries(3))
        .build();
```

`timeout` is in **seconds**. The full option set, and what is and is not defaulted, is in
**java-configuration-resilience** — read it before assuming retries are on.

## Choosing the environment / base URL

Environments are members of an `enum Environment` in the SDK's root package. **Read the enum for the real
member names before naming one.**

Do not assume a particular member exists — there may
be no `PRODUCTION` at all. A name also does **not** imply a live host: match each member to the URL it
actually resolves to in the private `environmentMapper` method of `{Api}Client.java`.

The base URL is **derived** from the selected environment plus a `Server` enum member (and any server
parameters the spec declares). The default environment is
whatever the `Builder`'s `environment` field is initialized to; read it rather than assume.

```java
{Api}Client client = new {Api}Client.Builder()
        .environment(Environment.{MEMBER})
        .build();

String url = client.getBaseUri();                 // resolved URL for the default server
String url2 = client.getBaseUri(Server.{MEMBER}); // for a named server
```

Some SDKs expose server parameters (a port, a tenant id, a template variable) as extra `Builder`
methods that feed the base-URL template. To point the SDK at a mock or a proxy that no `Environment`
member covers, see **java-configuration-resilience**.

## Accessing controllers — accessors on the client

Unlike some SDKs, you do not construct controllers yourself. The client exposes one accessor per API
group, named `get` + the controller's class name:

```java
{Controller} controller = client.get{Controller}();
```

The class name — and therefore the accessor — is named per SDK: the group name plus a
postfix that defaults to `Controller`, but which may be something else (`...Api`) or absent entirely.
**Read the accessor names off `{Api}Client.java`, or off `doc/client.md`, rather than guessing.**
Controllers are created once in the client constructor and returned by reference; they are stateless
wrappers, so holding one is fine.

Because this SDK ships no generated interfaces, the accessor's declared type is the `final`
implementation class itself; there is no interface to hold instead.

## Cloning a configured client

`client.newBuilder()` returns a `Builder` pre-populated with the current client's state — the
environment, any server parameters (a `region`/tenant/port setter), the HTTP client, the callback and the
http-client configuration, the logging configuration, and **every auth credential model**. Read
`newBuilder()` in `{Api}Client.java` for the exact set. Use it to produce a variant rather than
rebuilding from scratch:

```java
{Api}Client withLongerTimeout = client.newBuilder()
        .httpClientConfig(configBuilder -> configBuilder.timeout(120))
        .build();
```

This is also the documented way to attach a freshly fetched OAuth token — see **java-authentication**.

## Client lifetime, reuse, and shutdown

Build the client **once at startup and reuse it** for the process lifetime. Each `build()` creates a new
underlying OkHttp client wrapper and re-registers the SDK's interceptors. The dispatcher and connection
pool are shared process-wide (that is why `shutdown()` is static), so pooling survives — but building
per request still discards any cached OAuth token and burns allocations for nothing.

`shutdown()` is **`public static void`** on the client class — it shuts down the shared OkHttp
dispatcher and connection pool for the whole SDK, not just one instance. Call it once, on the class, when
the process is winding down:

```java
{Api}Client.shutdown();
```

Writing `client.shutdown()` compiles, but it is a static call through an instance and reads as if it
affected only that client. It does not — prefer the class form.

## Dependency injection

The client is immutable once built, so a singleton is the right scope everywhere. Inject the `{Api}Client` (or a narrow interface of your own over the operations you use)
rather than building one inside consumers.


## Next

- Configure authentication → **java-authentication**
- Make your first call → **java-calling-endpoints**
- Tune retries/timeouts/proxy → **java-configuration-resilience**
