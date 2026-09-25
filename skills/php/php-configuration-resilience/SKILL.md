---
name: 'php-configuration-resilience'
description: 'Tune the PayPal Server SDK PHP SDK client — retries, timeouts, proxy, logging and the base URL. Load before changing any transport setting. The setter list won''t tell you retrying takes three switches rather than one, that `maximumRetryWaitTime` is a cumulative budget rather than a cap, or that pagination is entirely manual here.'
---

# Configuration & resilience for an APIMatic PHP SDK

Everything is set on the client builder at construction time (see **php-client-initialization**). There
is no per-request options argument and no way to reach the underlying HTTP client.

```php
$client = {Client}Builder::init()
    ->environment(Environment::{CONSTANT})
    ->timeout(30)                       // SECONDS
    ->enableRetries(true)
    ->numberOfRetries(3)
    ->maximumRetryWaitTime(30)          // REQUIRED — see below; 0 retries nothing
    ->build();
```

> The `maximumRetryWaitTime` line is not optional. `MAXIMUM_RETRY_WAIT_TIME` is generated per SDK and is
> commonly `0`, and a budget of `0` fits no backoff interval, so
> `enableRetries(true)` plus `numberOfRetries(3)` without it retries **nothing**. Check the constant in
> `src/ConfigurationDefaults.php`.

## Read the real defaults — they are generated, not universal

`src/ConfigurationDefaults.php` holds a `public const` for every option. **Those constants are this
SDK's defaults**; they are fixed when the SDK is generated, so they differ from SDK to SDK. Read them
rather than assuming:

`TIMEOUT`, `ENABLE_RETRIES`, `NUMBER_OF_RETRIES`, `RETRY_INTERVAL`, `BACK_OFF_FACTOR`,
`MAXIMUM_RETRY_WAIT_TIME`, `RETRY_ON_TIMEOUT`, `HTTP_STATUS_CODES_TO_RETRY`, `HTTP_METHODS_TO_RETRY`,
`ENVIRONMENT`, `PROXY_CONFIGURATION`.

The one default that does **not** vary: `ENABLE_RETRIES` is always generated as `false`.

## Retries

The values below are an **illustrative policy you are choosing**, not this SDK's defaults — for those,
read `ConfigurationDefaults`:

```php
$client = {Client}Builder::init()
    ->enableRetries(true)          // master switch — false in every generated SDK
    ->numberOfRetries(3)
    ->retryInterval(1.0)           // seconds before the first retry
    ->backOffFactor(2.0)           // multiplier applied to the interval each attempt
    ->maximumRetryWaitTime(30)     // seconds — a BUDGET that must cover the waits, not a cap
    ->retryOnTimeout(true)
    ->httpStatusCodesToRetry([408, 429, 500, 502, 503, 504])
    ->httpMethodsToRetry(['GET', 'PUT'])
    ->build();
```

Those eight setters are the complete retry surface. Things the names do not tell you:

- **Three switches, not one.** `enableRetries(true)` alone does nothing if `NUMBER_OF_RETRIES` was
  generated as `0`; a non-zero `numberOfRetries` does nothing while retries are disabled; and **both** do
  nothing while `maximumRetryWaitTime` is too small. `NUMBER_OF_RETRIES` and `MAXIMUM_RETRY_WAIT_TIME`
  are both commonly `0` as well, so expect to set all three by hand. Check all three constants in
  `ConfigurationDefaults`.
- **`maximumRetryWaitTime` is a cumulative budget, not a cap.** The HTTP client computes the next wait as
  `RETRY_INTERVAL * BACK_OFF_FACTOR^attempt` plus up to ~0.1 s of jitter, and schedules the retry only
  while that wait still fits the remaining budget. So the value must be at least the **sum** of the waits
  you want — `RETRY_INTERVAL * (BACK_OFF_FACTOR^0 + … + BACK_OFF_FACTOR^(NUMBER_OF_RETRIES-1))` plus the
  jitter — and anything below the first interval yields **zero** retries however the other two are set.
  `MAXIMUM_RETRY_WAIT_TIME` is generated per SDK, so read the constant rather
  than assuming a generous default.
- **Only the methods in `httpMethodsToRetry` retry.** The generated set is typically the idempotent ones,
  so `POST`, `PATCH` and `DELETE` failures usually surface with no second attempt. Add one only if that
  operation is genuinely idempotent — and prefer an idempotency key if the API offers one.
- **`timeout` bounds an attempt, not the call.** With retries on, a call's worst-case wall time is
  several times the timeout plus the backoff intervals; `maximumRetryWaitTime` bounds only the waiting,
  not the attempts. There is no per-call cancellation mechanism.
- `retryOnTimeout` decides whether a timed-out attempt counts as retryable at all, independently of the
  status-code list.
- **Retries are off in every generated PHP SDK, whatever the other knobs say.** `ENABLE_RETRIES` is the
  flag the client reads before retrying anything, so an SDK can ship `NUMBER_OF_RETRIES = 3` and still
  retry nothing; `numberOfRetries(0)` is the wrong lever, and there is nothing to disable. Switch them
  on with all three switches above, and only where nothing above the
  SDK already retries — a queue worker, a job runner, a failover wrapper, or your own orchestration
  loop. Retry layers multiply rather than add: three retries is **four** requests per attempt, so
  inside a 3-attempt job it is twelve requests against an API whose rate limit counts every one.

## Timeouts

`timeout(int $seconds)` — **seconds**, not milliseconds. It is the only timeout knob; connect and read
timeouts are not separately configurable.

> ### ⚠ Read `TIMEOUT` before you assume there is one — `0` means *no* timeout
>
> `ConfigurationDefaults::TIMEOUT` is generated per SDK, so it
> varies between SDKs and **`0` is what an SDK built without a timeout carries**. That value is not "use a sensible
> default": it reaches cURL as `CURLOPT_TIMEOUT`, where `0` means *never time out*. A client built
> without calling `timeout(...)` on such an SDK waits indefinitely on a provider that accepts the
> connection and then stops responding.
>
> Open `src/ConfigurationDefaults.php` and read the constant. If it is `0`, **set `timeout(...)`
> explicitly on every client you construct** — the same way you would `enableRetries(...)`. This is the
> more dangerous of the two defaults: an SDK that does not retry fails fast and visibly, while one with
> no timeout hangs.

> **A call that fetches a token spends `timeout` twice.** This applies to an SDK secured by an OAuth
> grant — check `src/Authentication/` for an OAuth manager; if there is none, skip this.
>
> An operation whose cached token is missing or expired fetches one first, as a **separate request** on
> the same client and the same `timeout` — which, per the warning above, may be no bound at all when
> `ConfigurationDefaults::TIMEOUT` is `0`. `timeout` is per request, so the operation can take up to
> **two** full periods; size any caller-side deadline against two, not one, and note there is no
> per-call override to shorten either of them.
>
> **A token-fetch failure reaches you disguised as bad credentials.** The manager swallows the error and
> hands back the token it already held (`null` on a first call), so a token endpoint that times out,
> refuses the connection or returns a `500` produces the same `'Client is not authorized...'`
> `InvalidArgumentException` a wrong client id does. Call `fetchToken()` yourself at startup, where the
> real exception is still in flight, or configure a token provider.
>
> **`fetchToken()` throws a bare `\InvalidArgumentException` too — not the typed OAuth exception.** On a
> wrapper build the token operation's handler chain ends `->returnApiResponse()`, and the runtime's
> response-error path returns the wrapper *before* it reaches any throw — so a typed OAuth exception
> class that the same chain registers for the credential-failure statuses is never constructed. The
> manager detects the failure itself and raises `\InvalidArgumentException` with the provider's payload
> **serialized into the message string**, which is a different message from the two above and comes from
> a different method. Three consequences worth planning for, none of them visible in the class list:
> `catch` on the typed class never matches; the exception is not an `ApiException`, so a ladder built on
> the SDK's own base class misses it; and `error`/`error_description` survive only as text inside
> `getMessage()`. Check the registered class against `returnApiResponse()` before writing the catch —
> the same rule as everywhere else in this SDK — and if you need the fields structured, read them off
> the wrapper by calling the token operation through its controller instead.
>
> **The provider is not on the client builder.** It is `oAuthTokenProvider(callable)` on the scheme's
> *credentials* builder in `src/Authentication/` — the file whose name ends `CredentialsBuilder`.

## Base URL and environment

There is **no free-form base-URL option.** The URL comes from the `Environment::` constant you select,
looked up in a private map inside `src/{Client}.php`. Read `src/Environment.php` for the real constant
names — a `PRODUCTION` may not exist.

Some APIs declare server parameters (a templated host segment, a port); those become their own builder
setters. Check `src/{Client}Builder.php`. Confirm what you actually got with `$client->getBaseUri()`.

If no environment points where you need — a local mock, a gateway, a recording proxy — **the generated
builder cannot get you there.** `src/{Client}Builder.php` exposes no HTTP-client injection point and no
URL setter, so there is nothing on it to turn. **php-testing** says the same from the testing side.

**The core client underneath it can, at a price.** `apimatic/core`'s `ClientBuilder` is public API and
takes both an HTTP client and `serverUrls(...)`, and the generated base controller's constructor is
public — so the base URL *is* reachable by assembling the core client yourself. But then you own
everything the generated builder was configuring: auth managers, the user agent, global errors, the
retry and timeout configuration. Whatever you do not wire up is simply absent, and the omission does not
announce itself. It is also a construction path the generated code does not take, so a regeneration can
move it under you.

Order these by what your target actually is. `proxyConfiguration(...)` wants a real HTTP proxy — against
an `https://` target cURL issues `CONNECT`, which a plain mock cannot terminate — and DNS can move a
host but not a **scheme or a port**. So for the most common reason to redirect at all, pointing at a
local mock or a gateway on another port, neither reaches and assembling the core client yourself is the
answer rather than the last resort. Prefer proxy or DNS where the target is the same scheme and port on
a different host; it is the smaller commitment when it fits.

## Proxy

```php
use PaypalServerSdkLib\Proxy\ProxyConfigurationBuilder;

$client = {Client}Builder::init()
    ->proxyConfiguration(
        ProxyConfigurationBuilder::init('http://localhost')
            ->port(8080)
            ->auth('username', 'password')
            ->authMethod(CURLAUTH_BASIC)
            ->tunnel(true)
    )
    ->build();
```

`init(...)` takes the address; everything else is optional. `authMethod` takes a cURL constant.

## Pagination — entirely manual
**This API declares no paginated operation, and PHP SDKs generate no pagination support in any case**: no
iterator, no page wrapper, no `paginate()`, no auto-advance. A list endpoint is an ordinary operation; if
it takes `page` / `offset` / `cursor` / `limit` parameters, it returns one page and you drive it yourself,
stopping on the API's own end signal:

Every operation in this SDK returns `ApiResponse`, so the page model is behind `$result->getResult()`.

**The same check settles how you pass the paging arguments.** An operation with more than one parameter
is generated either positionally or as a single associative `array $options`, decided by the build's
parameter-collapsing setting — so read the signature rather than copying either form below.

```php
$page = 1;
$size = 100;

do {
    // positional build:
    $result = $controller->{operation}($page, $size);
    // collapsed build — the same call:
    // $result = $controller->{operation}(['{pageParam}' => $page, '{sizeParam}' => $size]);

    $items  = $result->getResult()->get{ItemsField}();

    foreach ($items as $item) {
        // …
    }

    $page++;
} while (count($items) === $size);
```

For a cursor-based endpoint, read the next cursor off the page model and loop until it is absent. Check
the response model in `src/Models/` for whatever "there is more" field the API actually returns — a
short page is a heuristic, not a contract.

## Logging
**This SDK has logging built in**: `src/Logging/LoggingConfigurationBuilder.php` exists
and the client builder has a `loggingConfiguration(…)` setter. Nothing is logged until you call it:

```php
use PaypalServerSdkLib\Logging\LoggingConfigurationBuilder;
use PaypalServerSdkLib\Logging\RequestLoggingConfigurationBuilder;
use PaypalServerSdkLib\Logging\ResponseLoggingConfigurationBuilder;
use Psr\Log\LogLevel;

$client = {Client}Builder::init()
    ->loggingConfiguration(
        LoggingConfigurationBuilder::init()
            ->logger($psr3Logger)             // any Psr\Log\LoggerInterface
            ->level(LogLevel::DEBUG)
            ->maskSensitiveHeaders(true)
            ->requestConfiguration(
                RequestLoggingConfigurationBuilder::init()
                    ->body(true)
                    ->headers(true)
                    ->includeQueryInPath(true)
            )
            ->responseConfiguration(
                ResponseLoggingConfigurationBuilder::init()
                    ->body(true)
                    ->headers(true)
            )
    )
    ->build();
```

- **Nothing is logged until you call `loggingConfiguration(…)`** — the client only builds a logging
  configuration when that setter received a `LoggingConfigurationBuilder`. Once it has one, the
  defaults are a console logger at `LogLevel::INFO` with request and response bodies and headers all
  **off** and `maskSensitiveHeaders` **on**, so you get a line per call and no payloads until you ask
  for them.
- `level()` accepts only PSR-3 `LogLevel` values and throws `Psr\Log\InvalidArgumentException` otherwise.
- Both request and response configurations also take `includeHeaders(...)`, `excludeHeaders(...)` and
  `unmaskHeaders(...)` as variadic header-name lists.

## Other conditional options

`skipSslVerification(bool)`, `additionalHeaders(array)` and `userAgentDetail(string)` are each generated
only in an SDK built with them. Their absence from `src/{Client}Builder.php` means this
SDK does not support them — do not work around it by editing the SDK.

## Next

- Step 7, stubbing the SDK → **php-testing**
