---
name: 'java-configuration-resilience'
description: 'Tune the PayPal Server SDK Java SDK client — retries, timeouts, transport, logging and the base URL. Load before changing any transport setting. The option list won''t tell you a client built without `timeout(...)` waits forever, that the verb list does not stop a `POST` being re-sent, or what a rewriting interceptor does once retries are on.'
---

# Configuration & resilience for an APIMatic Java SDK

All configuration happens when you build the client (see **java-client-initialization**). Transport
tuning is nested inside the `httpClientConfig` **lambda** — it is not on the client `Builder` directly:

```java
import {rootPackage}.{Api}Client;
import {rootPackage}.Environment;

{Api}Client client = new {Api}Client.Builder()
        .environment(Environment.{MEMBER})
        .httpClientConfig(configBuilder -> configBuilder
                .timeout(30)                       // SECONDS
                .numberOfRetries(3)
                .backOffFactor(2)
                .retryInterval(1)
                .shouldRetryOnTimeout(true))
        .build();
```

## The full `HttpClientConfiguration.Builder` surface

| Method | Type | Meaning |
| --- | --- | --- |
| `timeout(long)` | seconds | per-attempt request timeout — the generated Javadoc says *"The timeout in seconds"* |
| `numberOfRetries(int)` | count | retries **after** the first attempt |
| `backOffFactor(int)` | multiplier | *"to use in calculation of wait time for next request in case of failure"* |
| `retryInterval(long)` | | the other half of that calculation — the base wait |
| `maximumRetryWaitTime(long)` | | *"the maximum wait time for overall retrying requests"* |
| `shouldRetryOnTimeout(boolean)` | | whether a timed-out attempt is retried |
| `httpStatusCodesToRetry(Set<Integer>)` | | which statuses are retryable |
| `httpMethodsToRetry(Set<HttpMethod>)` | | which HTTP methods are retryable |
| `httpClientInstance(okhttp3.OkHttpClient)` | | use your own OkHttp client |
| `httpClientInstance(okhttp3.OkHttpClient, boolean overrideHttpClientConfigurations)` | | as above, and whether the SDK may override its timeout/retry settings |
| `proxyConfig(HttpProxyConfiguration.Builder)` | | route through a proxy |

Read the same list back off a built client with `client.getHttpClientConfig()`, which returns a
`ReadonlyHttpClientConfiguration` with one accessor per option — `get...` for the values, but
`shouldRetryOnTimeout()` and `shouldOverrideHttpClientConfigurations()` for the two booleans. Read
`<root>/http/client/ReadonlyHttpClientConfiguration.java` for the exact list. It is the fastest way to
confirm what a running client is actually using.

**Every one of these is client-wide.** Operations take no request-options argument and no cancellation
handle of any kind — `{operation}Async` returns a `CompletableFuture`, but cancelling that future does
not cancel the HTTP call. If you need a deadline a caller can set per request, it has to live in your
own code above the SDK.

> **A call that fetches a token spends that budget twice.** This applies to an SDK secured by an OAuth
> grant — check the auth setters on the client `Builder`; if there is no OAuth model, skip this.
>
> An operation whose cached token is missing or expired fetches one first: the **token request** goes
> out on the same OkHttp client, under the same `timeout` — which, per the warning below, may be no
> bound at all. `timeout` is per request, so the operation can take up to **two** full periods; size a
> caller-side deadline against two, not one. The fetch is synchronous on the calling thread even for
> `{operation}Async`, because the request is built before the async handoff.
>
> **A token-fetch failure arrives in the words a wrong client id produces.** The manager swallows the
> underlying exception, returns the token it already held (`null` on a first call), and what surfaces is
> an `AuthValidationException` about missing authorization. To tell "the provider is down" from "our
> credentials are wrong", call `fetchToken()` yourself at startup where the real exception is still in
> flight, or configure an OAuth token provider, which never reaches that path.

## Retry defaults — retries are off, and the generated code cannot turn them on

The generated `Builder()` constructor in `<root>/http/client/HttpClientConfiguration.java` sets
**exactly two** things:
`httpStatusCodesToRetry(...)` and `httpMethodsToRetry(...)` — both generated per SDK, so they vary
between SDKs. Read the constructor for the real lists.

**Everything else — including `numberOfRetries` — is left at the runtime's own default**, which lives in
the `io.apimatic:core` dependency, not in the generated code, and for `numberOfRetries` that default is
`0`.

> ### ⚠ `timeout` is left at the runtime default too, and that default is *no timeout*
>
> **A client you build without calling `timeout(...)` waits forever.** The runtime's default is `0`
> seconds, and the adapter passes it straight to OkHttp's `readTimeout`, `writeTimeout` and
> `connectTimeout` — where **`0` means no limit**, not "use a sensible one". With `numberOfRetries` at
> its own default of `0`, `callTimeout` receives the same `0`, so nothing bounds the call at any layer.
>
> A provider that accepts the connection and then stops responding will hold the calling thread until
> the socket is closed from the other end, which may be never. This is the more dangerous of the two
> defaults: an SDK that does not retry fails fast and visibly, while one with no timeout hangs.
>
> Confirm rather than assume — the adapter version floats independently of the SDK. Build a client the
> way production builds it and read `client.getHttpClientConfig().getTimeout()`. **Set `timeout(...)`
> explicitly on every client you construct**, the same way you would set `numberOfRetries(...)`.

Notes that follow from the same shape:

- Only the methods in `httpMethodsToRetry` are retried **when the retry decision is the interceptor's
  to make** — that is, for a status code, or for the interceptor's own timeout. That set typically
  covers idempotent verbs; add one only when the operation is genuinely idempotent.

> ### ⚠ `httpMethodsToRetry` does not stop a `POST` being re-sent
>
> Two paths re-send a request **without consulting the verb list at all**, so a write can be repeated
> even when `POST` is absent from it.
>
> **1. The adapter re-sends on any `SocketException`, unbounded.** Its retry interceptor catches one and
> calls itself — a **recursive** re-send that consults neither `httpMethodsToRetry` nor
> `numberOfRetries`. `SocketException` covers connection reset, broken pipe and "software caused
> connection abort", all of which happen **after** the request bytes are on the wire, so the server may
> well have processed the write. This path is installed only when `numberOfRetries > 0` — so it is
> dormant at the default and switches on, unbounded, the moment anyone enables retries at all.
>
> **2. OkHttp's own connection recovery is on by default, and it never looks at the verb.** Every
> client the adapter *constructs for you* sets `retryOnConnectionFailure(true)`, and OkHttp's recovery
> checks the failure kind and the remaining routes only. Unlike point 1 this one **is** avoidable: the
> adapter derives from `okHttpClient.newBuilder()`, so a client you pass to `httpClientInstance(...)`
> keeps `retryOnConnectionFailure(false)` if you set it. That is the only way to turn this arm off.
>
> So on this stack, `numberOfRetries` is not a bound on writes and `httpMethodsToRetry` is not a filter
> on them. If an operation must never be sent twice, neither setting expresses that: give writes their
> own client with retries off, supply your own `OkHttpClient` with `retryOnConnectionFailure(false)`, or
> make the operation idempotent at the provider with a caller-supplied key.
- `timeout` bounds a **single attempt** (connect/read/write), not the whole call. As soon as
  `numberOfRetries > 0` the runtime sets OkHttp's whole-call timeout to `maximumRetryWaitTime` *instead
  of* `timeout`, so `maximumRetryWaitTime` — not `timeout` — is the real wall-clock ceiling for the
  operation, retries and backoff included. Read its value off
  `client.getHttpClientConfig().getMaximumRetryWaitTime()` rather than assuming a number, and raise it if
  `(numberOfRetries + 1) × timeout` plus backoff could legitimately exceed it.
- Retries happen inside the SDK, before any exception reaches your `catch` block.
- **`numberOfRetries(0)` is a no-op.** The OkHttp adapter installs its retry interceptor **only when
  `numberOfRetries` is above zero**, so at the default a client makes exactly one attempt per call, and
  `numberOfRetries(n)` in your own `httpClientConfig` lambda is the only thing that switches retrying
  on. Once you set it, every status and method listed
  in that constructor is retried underneath your code. Raise it only where nothing above the SDK already
  retries — a job scheduler, a message consumer, a failover wrapper, or your own loop. Retry layers
  multiply rather than add: `numberOfRetries(3)` is **four** requests per attempt, so inside a
  3-attempt job it is twelve requests against an API whose rate limit counts every one.

## Base URL / environment

There is **no free-form base-URL option**. The URL is derived from the selected `Environment` member and
a `Server` member by a private resolver in `{Api}Client.java`; some SDKs additionally expose server
parameters (a port, a tenant) as their own `Builder` methods.

```java
{Api}Client client = new {Api}Client.Builder()
        .environment(Environment.{MEMBER})
        .build();

client.getBaseUri();                  // what it actually resolved to
```

Read the `Environment` enum for the real member names — a
`PRODUCTION` member may not exist.

To point the SDK somewhere no environment covers (a mock server, a recording proxy, a gateway), route
through a proxy (below), or inject an `OkHttpClient` carrying an interceptor that rewrites the URL and
then forwards it:

```java
Interceptor rewrite = chain -> {
    HttpUrl target = HttpUrl.get(System.getenv("API_BASE_URL"));
    HttpUrl url = chain.request().url().newBuilder()
            .scheme(target.scheme()).host(target.host()).port(target.port())
            .build();
    return chain.proceed(chain.request().newBuilder().url(url).build());
};
```

> ### ⚠ That interceptor and the SDK's retries cannot both be on
>
> **With `numberOfRetries` above zero, a rewriting interceptor makes every call fail with a
> `NullPointerException` raised before any network I/O.**
>
> The adapter's retry interceptor loses its per-request state when it is handed a **rebuilt** request,
> which is what a rewrite does, and dereferences null.
> Ordering makes it unavoidable rather than a matter of care: the SDK wraps the client
> you supply with `okHttpClient.newBuilder()`, so your interceptors are already ahead of the one it adds
> and your rewrite always runs first.
>
> The retry interceptor is installed **only when `numberOfRetries > 0`**, which is why this can lie
> dormant: redirect the base URL today with retries off, raise `numberOfRetries` next week, and the
> failure arrives pointing at code you did not write. If you need both, keep retries at `0` on the
> redirected client and put the retry loop in your own code above the SDK.
>
> Verify the version you actually have before designing around this: the adapter floats independently
> of the SDK.

## Proxy

```java
.httpClientConfig(configBuilder -> configBuilder
        .proxyConfig(new HttpProxyConfiguration.Builder("proxy.internal", 8080)
                .auth(System.getenv("PROXY_USER"), System.getenv("PROXY_PASSWORD"))))
```

Address and port are constructor arguments; `auth(username, password)` is the only optional setter.

## Bringing your own OkHttp client

```java
OkHttpClient shared = new OkHttpClient.Builder()
        .connectionPool(new ConnectionPool(20, 5, TimeUnit.MINUTES))
        .addInterceptor(myInterceptor)
        .build();

.httpClientConfig(configBuilder -> configBuilder
        .httpClientInstance(shared, true))   // true = let the SDK apply its timeout/retry settings
```

The one-argument overload leaves `overrideHttpClientConfigurations` at its default of **true** — i.e. the
SDK still rebuilds your client with its own timeouts and interceptors. To keep your client's own settings
you must pass the two-argument form explicitly: `.httpClientInstance(shared, false)`. This is also the
seam for anything OkHttp can do that the SDK does not expose: custom TLS, connection pooling, or an
interceptor that logs or rewrites requests.

Remember that `{Api}Client.shutdown()` is static and shuts down the SDK's OkHttp resources — if you own
the client instance, manage its lifecycle yourself too.

## Pagination — none in this SDK

**This API declares no paginated operation**, so the SDK ships no `<root>/utilities/pagination/` package
and no operation returns a `PagedIterable`/`PagedFlux`. Every list endpoint is a plain call: drive its own
page/offset/cursor parameters yourself and stop when a page comes back shorter than you asked for.

## Observing requests and responses

**Always available: `HttpCallback`.** The client `Builder` accepts one, and it fires around every call:

```java
{Api}Client client = new {Api}Client.Builder()
        .httpCallback(new HttpCallback() {
            @Override
            public void onBeforeRequest(Request request) {
                log.debug("{} {}", request.getHttpMethod(), request.getQueryUrl());
            }

            @Override
            public void onAfterResponse(Context context) {
                log.debug("-> {}", context.getResponse().getStatusCode());
            }
        })
        .build();
```

`HttpCallback` extends the runtime's `Callback`, so its parameters are the **runtime interfaces**
`Request` and `Context` — **not** the SDK's own `HttpRequest` / `HttpResponse`. **No cast is needed**:
`Request` itself declares `getHttpMethod()`, `getQueryUrl()`, `getHeaders()` and `getBody()`, and
`context.getResponse()` declares `getStatusCode()`, `getHeaders()`, `getBody()`, `getRawBody()` and
`getRawBodyString()`. Cast to the SDK's `HttpRequest`/`HttpResponse` only when you want its covariant
return types — `Headers` instead of `HttpHeaders`, `HttpMethod` instead of `Method`. Take the imports for
`Request` and `Context` from the `Callback` interface that the SDK's `HttpCallback` extends.

**Also available here: built-in logging.** This SDK **ships logging**, so it has a
`<root>/logging/configuration/` package and a `loggingConfig(...)` method on the client `Builder`:

```java
.loggingConfig(logBuilder -> logBuilder
        .level(org.slf4j.event.Level.INFO)
        .maskSensitiveHeaders(true)
        .requestConfig(req -> req.body(true).headers(true).excludeHeaders("authorization"))
        .responseConfig(res -> res.body(true).headers(false)))
```

There is also a no-argument `loggingConfig()` that turns on the default console logger. The default
logger writes to stdout and needs nothing extra on the classpath.

> **Two different methods turn on the default logger, on two different types — do not look for either on
> the other.** `loggingConfig()` with no argument is on the **client `Builder`** and is the whole
> statement. `useDefaultLogger()` is on the **logging-configuration builder**, so it is only reachable
> *inside* the lambda, where it is what you combine with `.level(...)` or `.maskSensitiveHeaders(...)` to
> keep the default sink while changing something else. Reaching for the wrong one is a compile error that
> reads as if the method does not exist at all.

To route logs into your application's
logging stack instead, pass your own SLF4J logger with `.logger(LoggerFactory.getLogger(...))` inside the
`loggingConfig` lambda — that path, and only that path, needs an SLF4J binding. Read
`<root>/logging/configuration/ReadonlyLoggingConfiguration.java` for the full option set.

## Next

- Step 7, stubbing the SDK → **java-testing**
