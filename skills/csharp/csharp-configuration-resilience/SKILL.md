---
name: 'csharp-configuration-resilience'
description: 'Tune the PayPal Server SDK C# SDK client — retries, timeouts, proxy, transport, logging and the base URL. Load before changing any transport setting. The option list won''t tell you nothing retries until you raise the count yourself, that the verb whitelist does not cover a call that fails by throwing, or how to reach a host no `Environment` member covers.'
---

# Configuration & resilience for an APIMatic C# SDK

Everything is configured on the client `Builder` at construction (see
**csharp-client-initialization**). Transport tuning is **nested inside the `HttpClientConfig`
action** — it is not on the client `Builder` directly:

```csharp
var client = new PaypalServerSdkClient.Builder()
    .Environment(PaypalServerSdk.Standard.Environment.{Member})
    .HttpClientConfig(config => config
        .Timeout(TimeSpan.FromSeconds(30))       // TimeSpan — not an int of seconds
        .NumberOfRetries(3)
        .MaximumRetryWaitTime(TimeSpan.FromSeconds(30)))
    .Build();
```

Those values are **a policy you are choosing**, not this SDK's defaults. There is no
`PaypalServerSdkClient.Builder.Timeout(...)`, whatever `doc/client.md` lists — see
**csharp-client-initialization** for why that table is not a contract.

## The `HttpClientConfiguration.Builder` surface

| Method | Type | Meaning |
| --- | --- | --- |
| `Timeout(TimeSpan)` | `TimeSpan` | http client timeout |
| `NumberOfRetries(int)` | count | times a request is retried |
| `BackoffFactor(int)` | multiplier | exponential backoff between retry calls |
| `RetryInterval(double)` | `double` | interval between the endpoint calls |
| `MaximumRetryWaitTime(TimeSpan)` | `TimeSpan` | cap on the whole retried call, not just the waiting |
| `StatusCodesToRetry(IList<int>)` | statuses | which statuses invoke a retry |
| `RequestMethodsToRetry(IList<HttpMethod>)` | `System.Net.Http.HttpMethod` | which verbs invoke a retry |
| `HttpClientInstance(HttpClient, bool overrideHttpClientConfiguration = true)` | | use your own client — the getter is **never `null`**, since the runtime materialises one when you inject nothing, so assert identity (`NotSame`) rather than nullness when checking whether yours survived |
| `Proxy(ProxyConfigurationBuilder)` | | route through a proxy |

`client.HttpClientConfiguration` — read-only on the **built** client — reads them back: a getter for
every row above **except `Proxy`**, plus `OverrideHttpClientConfiguration`.

## Retry defaults — read them, do not assume

`Http/Client/HttpClientConfiguration.cs` sets **no retry, timeout or status-code default of its own**:
its `Builder` wraps `CoreHttpClientConfiguration.Builder` and every setter forwards, so every
effective default comes from the `APIMatic.Core` package, whose version the `.csproj` floats. That
default is `NumberOfRetries = 0`, so no verb is retried — `GET` and `PUT` included — until you raise the
count yourself.

> Do not assume a default, and do not carry one over from another SDK. Build a client the way production
> builds it, then read `NumberOfRetries`, `StatusCodesToRetry` and `RequestMethodsToRetry` off it
> individually — its `ToString()` prints unlabelled positional values and renders collections as their
> type name, so it is not usable for this.

> ### ⚠ `RequestMethodsToRetry` does not protect a `POST`
>
> **The verb whitelist gates only the *response*-triggered arm of the retry policy. A call that fails by
> throwing is retried whatever its verb.** The runtime's policy has one arm that consults
> `RequestMethodsToRetry` and two — a cancelled task, and a failed HTTP request — that do not.
>
> So with `NumberOfRetries(2)` and a whitelist of `GET` alone, a creating `POST` that times out or drops
> its connection **is sent again** — which is the duplicate-write hazard, arriving through the one door
> the whitelist looks like it closes. Whether that becomes two records depends on whether the operation
> is idempotent server-side, which is a property of the API you are calling.
>
> A raised `NumberOfRetries` is therefore a decision about **every** verb the client sends, not just the
> whitelisted ones. If some operations must never be re-sent, the whitelist will not express that —
> separate the clients, or keep `NumberOfRetries(0)` on the one that performs writes and raise it only
> on the read path.

- Once you do raise it, the verbs in `RequestMethodsToRetry` are what the **status-code** arm will retry;
  that list defaults to **`GET, PUT`** on every build checked, so a `POST` is not retried *on a 5xx*
  until you add it — subject to the exception arm above, which ignores the list entirely.
  **Check what your SDK's surface actually is before tuning retries at all** —
  `grep -rhoE 'Setup\(HttpMethod\.[A-Za-z]+|Setup\(new HttpMethod\("[A-Z]+"' Controllers/ | sort | uniq -c`
  (the folder name varies per SDK, so take it from the token rather than assuming) — and match
  **both** forms, because verbs with no `HttpMethod` static (`PATCH` among them) are emitted as
  `new HttpMethod("PATCH")` and a census that looks only for `HttpMethod.` silently under-reports them.
  On a write-heavy API almost every operation can be `POST`, and raising `NumberOfRetries` alone then
  changes nothing.
- **`Timeout` bounds one attempt** — it is the underlying `HttpClient`'s timeout, so a retried call
  can run for a multiple of it. `MaximumRetryWaitTime` is the ceiling on the whole retried sequence;
  size a caller-side deadline from that. **Breaching it does not raise a cancellation exception** —
  what surfaces is a Polly timeout type, so a `catch` written for `OperationCanceledException` will not
  see it (see **csharp-error-handling**). Retries happen inside the SDK, before any exception reaches
  your `catch` — see **csharp-error-handling**.
- Operation XML doc comments sometimes assert retry behaviour ("GET is idempotent, so the SDK retries
  it by default on 408, 429 and 5xx"). **That prose is copied from the spec's description**, and the
  `*Default*:` notes in `doc/client.md` are documentation too. Neither is evidence of this build.
- **No generated file names a retry value**, so the count is left at the runtime's own default until
  you call `.NumberOfRetries(...)` yourself — and
  only where nothing above the SDK already retries — a hosted service, a message consumer, a Polly
  policy, or your own orchestration loop. Retry layers multiply rather than add: three retries is
  **four** requests per attempt, so inside a 3-attempt job it is twelve requests against an API whose
  rate limit counts every one. To pin one attempt regardless of what anyone later configures, pass
  `.HttpClientInstance(client, overrideHttpClientConfiguration: false)` — that bypasses the SDK's retry
  settings entirely.

## Timeouts and cancellation

`Timeout(TimeSpan)` is client-wide; there is no per-call timeout parameter. Per-call cancellation is a
trailing `CancellationToken` that **only the `{Operation}Async` overloads take** — see
**csharp-calling-endpoints** for the two method shapes and which one drops the token.

> ### ⚠ Your `CancellationToken` does not reach a token exchange
>
> **This applies to a scheme that fetches a token** — an OAuth grant. If the auth setters listed in
> **csharp-getting-started** show only static credentials, nothing below happens on your calls.
>
> An operation whose cached token is missing or expired fetches one **first**, as a separate HTTP
> request, and the token you passed does not travel with it: the auth manager calls the OAuth
> controller's token method without the trailing `CancellationToken`, so that request runs at
> `CancellationToken.None`. Your token still governs the operation itself.
>
> **`Timeout` does bound it**, which is the saving grace — the token request goes out on the same
> `HttpClient`. But `Timeout` is per request, so a call that has to fetch a token can take up to
> **two full `Timeout` periods**. Size a caller-side deadline against two, not one, and do not expect
> cancellation to shorten the first of them.
## Base URL / environment

There is **no free-form base-URL option** — neither the client `Builder` nor
`HttpClientConfiguration.Builder` has a `BaseUrl`, `BaseUri`, `ServerUrl` or `Endpoint` setter. The
URL comes from a `private static readonly` environments map inside `PaypalServerSdkClient.cs`, keyed by the
`Environment` member you chose and then by the `Server` alias the operation picks;
`client.GetBaseUri()` reports what it resolved to. **You do not choose the `Server`** — each operation
pins its own in its request builder (`.Server(Server.X)`), so `GetBaseUri(Server.X)` is a diagnostic and
not a routing control. Grep the controller for `.Server(` to see which one an operation pins; **no hits
means no operation pins one** and they all inherit the environment's default server. Server template variables declared by the spec
become **their own client-`Builder` methods**. Most are substituted into that URL string — but a
configuration variable can instead be a **global header sent on every request**, and one that defaults to
`string.Empty` ships blank rather than being omitted. Read the variable's registration in
`PaypalServerSdkClient.cs` to see which it is — see **csharp-client-initialization** for choosing among them.

To reach somewhere no environment covers — a local mock, a recording proxy, a gateway — route through a
proxy (below), or supply an `HttpClient` built over a **forwarding** `DelegatingHandler` that rewrites
`request.RequestUri` and then calls `base.SendAsync`:

```csharp
internal sealed class BaseUrlRewriteHandler : DelegatingHandler
{
    private readonly Uri _target;
    public BaseUrlRewriteHandler(Uri target, HttpMessageHandler inner) : base(inner) => _target = target;

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var b = new UriBuilder(request.RequestUri) { Scheme = _target.Scheme, Host = _target.Host, Port = _target.Port };
        request.RequestUri = b.Uri;                      // path and query preserved
        return base.SendAsync(request, cancellationToken);
    }
}
```

> **Do not reach for the stub handler in csharp-testing for this.** That one *answers* the request and
> never forwards it, which is what you want in a test and the opposite of what you want here. The two
> look alike and are not the same thing.

Two properties of this seam are worth knowing before you rely on it:

- **It sits below the SDK, so it catches every request the client makes — including the OAuth token
  exchange.** That is usually what you want: rewriting only the API host would leave the client
  authenticating against the original one. But it means a handler that filters by path has to account for
  the token endpoint deliberately rather than by omission.
- **Unlike replacing the transport wholesale, the SDK's own behaviour still runs.** Auth, the retry
  policy, the timeout and error mapping are all applied above the handler, so redirecting this way does
  not quietly opt you out of the rest of this skill.

## Proxy

```csharp
// ProxyConfigurationBuilder lives in PaypalServerSdk.Standard.Http.Client.Proxy
.HttpClientConfig(config => config
    .Proxy(new ProxyConfigurationBuilder("http://proxy.internal")
        .Port(8080)                              // the builder's own field default is 8080
        .Auth(user, pass)
        .Tunnel(true)))
```

The address is the constructor argument; `Port`, `Auth` and `Tunnel` are the only public setters, and
`Build()` is `internal` — hand the *builder* to `.Proxy(...)`. The method is `Proxy(...)`; the
generated docs call it `ProxyConfiguration(...)`, which does not exist.

## Bringing your own `HttpClient`

```csharp
.HttpClientConfig(config => config
    .HttpClientInstance(sharedHttpClient, overrideHttpClientConfiguration: true))
```

At the default `true` the SDK overwrites your instance's `Timeout` with its own and keeps its retry
policy. **Passing `false` takes the instance as-is and disables retrying entirely** — every
`NumberOfRetries`, `StatusCodesToRetry` and `RequestMethodsToRetry` value is then ignored. This is the
seam for what the SDK does not expose — a custom `HttpMessageHandler`, pooling, TLS, a handler that
logs or rewrites requests. **One `HttpClient` per built client, though** — at the default
`overrideHttpClientConfiguration: true` the SDK assigns `HttpClient.Timeout` during `Build()`, so passing an
instance that has already sent a request into a second `Build()` throws
`InvalidOperationException: This instance has already started one or more requests`.

**Re-apply `.HttpClientConfig(...)` on any client you clone**: `ToBuilder()` carries everything else
over but starts a fresh `HttpClientConfiguration.Builder` — see **csharp-client-initialization**.

## Pagination

**This API declares no paginated operation**, so the SDK ships no pagination utilities and no
operation returns a `Pageable<,>`. A list endpoint here returns the ordinary type: drive its own
page/cursor parameters yourself and stop on the API's end signal.

> The "ordinary type" above is the `ApiResponse<T>` envelope this SDK returns,
> so `Http/Response/ApiResponse.cs` is generated and every non-void, non-paginated operation returns it.

## Logging

Logging is built into this SDK — but it is **off until you ask for
it**: the builder's logging field starts `null`. `PaypalServerSdkClient.Builder` has a `LoggingConfig`
method and there is a `Logging/` folder beside `Http/`. The no-argument overload switches on the
built-in console logger; the action overload hands you a `LogBuilder`:

```csharp
.LoggingConfig(config => config
    .Logger(myLogger)                            // Microsoft.Extensions.Logging.ILogger
    .LogLevel(LogLevel.Information)
    .MaskSensitiveHeaders(true)
    .RequestConfig(req => req.Body(true).Headers(true).ExcludeHeaders("Authorization"))
    .ResponseConfig(res => res.Body(true).Headers(false)))
```

Both inner builders also take `IncludeHeaders(params string[])` and `UnmaskHeaders(params string[])`;
`IncludeQueryInPath(bool)` is request-only. Per the generated log-builder docs, an unconfigured logger
records **neither the body nor the headers**.

If `LoggingConfig` is absent, the SDK was generated without logging. Use `HttpCallback` — always
generated, and always **observation only**: it cannot alter or substitute a response. A bare instance
passed to `.HttpCallback(...)` already records the *last* exchange on its `Request` / `Response`
properties; subclass it and override the empty `OnBeforeRequest(HttpRequest)` /
`OnAfterResponse(HttpResponse)` hooks to log *every* exchange. `doc/http-request.md` and
`doc/http-response.md` list their members.

## The same options from configuration

Binding a section with `PaypalServerSdkClient.FromConfiguration(...)` reaches this surface too, under a
`HttpClientConfig` key (`LoggingConfig` for logging) — see
**csharp-client-initialization** for the binding itself. Three things fail quietly here: `HttpClientInstance` has no JSON form at all; the
proxy key is `Proxy`, matching the binder property, while the generated sample JSON writes
`ProxyConfiguration`, which binds to nothing; and `RequestMethodsToRetry` takes verb **strings**, an
unrecognised one being dropped without complaint. Config-driven logging always
installs the console logger, so only `.LoggingConfig(c => c.Logger(...))` in code can install your own
`ILogger`.

## Next

- Step 7, stubbing the SDK → **csharp-testing**
