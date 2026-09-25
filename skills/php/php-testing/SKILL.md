---
name: 'php-testing'
description: 'Unit-test code that calls the PayPal Server SDK PHP SDK. Load before stubbing the SDK. The class list won''t tell you which seam to fake, that nothing in the SDK is `final` so PHPUnit can double it, or that `httpCallback` observes a real call rather than replacing it.'
---

# Testing code that uses an APIMatic PHP SDK

**Start from the constraint:** the client builder has **no HTTP-client setter**, no handler stack, no
adapter and no base-URL option. Nothing in `src/` lets you swap the transport. Any advice that starts
"inject a mock HTTP client" does not apply to these SDKs — check `src/{Client}Builder.php` and you will
find only configuration values, credentials builders, `httpCallback` and `proxyConfiguration`.

So the seam is **above** the SDK, not inside it.

> Read the real accessor and class names from `src/{Client}.php` and the controller folder
> (`src/Controllers/` or `src/Apis/`, whichever `ls src/` shows); the class suffix is `Controller` on
> some builds and `Api` on others, so do not assume one.

The examples below use PHPUnit for reference only — mirror whatever the project already uses.

## Preferred seam — your own interface

Wrap the operations you actually use behind a narrow interface you own and fake that, so your tests
stay independent of SDK internals and survive regeneration. That gateway is also where error translation
belongs — it is the one place that knows how to unwrap an `ApiResponse` and what a failed one means.
Test your application against the interface, and the implementation against the SDK separately.

## Doubling the SDK classes directly

There are **no `final` classes and no `final` methods**, so PHPUnit can double a controller or
the client itself. `createMock()` does not invoke the constructor, which matters because the client's
constructor builds a real HTTP client:

```php
$controller = $this->createMock({Group}{Postfix}::class);
$controller->method('{operation}')->willReturn($apiResponseDouble);
// The stub must match the declared return type, so a bare array on a method typed
// `: ApiResponse` fails at stub time — build the double as shown further down.

$client = $this->createMock({Client}::class);
$client->method('get{Group}{Postfix}')->willReturn($controller);

$service = new MyService($client);
```

Build the expected models with their real builders (see **php-models**) so the shapes stay honest:

```php
$expectedModel = {Model}Builder::init(/* required fields */)->build();
```

### The error path — first, how does a failure surface?
**A bad status does not raise in this SDK.** Every operation returns the wrapper and a non-2xx comes back
inside it — a `willThrowException` stub for a 4xx/5xx exercises a path production code never takes. So stub
the **returned failure** for every status case.

One file is still worth checking, because it decides whether anything is throwable at all:
`src/Exceptions/ApiException.php`. Where it exists a **transport** failure still raises `ApiException`;
where it is absent nothing raises and the classes under `src/Exceptions/` are plain error models. Cover
both paths that your build actually has. See **php-error-handling**.

*Wherever `src/Exceptions/ApiException.php` exists — throw what the real SDK throws*
(see **php-error-handling**). Here that is the transport-failure test and nothing else: a non-2xx never
raises in this SDK. `ApiException`'s
constructor takes `(string $reason, HttpRequest $request, ?HttpResponse $response)`, so the cheapest
faithful stand-in for a transport failure is a `null` response — which is also the case whose
`getCode()` is `0`:

```php
$controller->method('{operation}')
    ->willThrowException(new ApiException('Boom', $this->createMock(HttpRequest::class), null));
```

There is no status-carrying `ApiException` to stub, because a non-2xx comes back inside the wrapper
instead — but the transport case is **not** the only throwing path.

> ### ⚠ An authenticated operation throws before it reaches your double
>
> **`AuthValidationException` is an `\InvalidArgumentException`, not an `ApiException`**, so a catch
> ladder built around `ApiException` does not catch it — and neither does a test that only asserts on
> the wrapper.
>
> It is thrown synchronously from inside every operation that declares auth, whenever credentials are
> missing or the token is expired. **A failed token exchange lands here too**: the OAuth manager
> swallows the fetch error and returns its held token, so a bad client secret leaves that token null and
> validation throws — the ordinary bad-credentials path, surfacing as an argument exception.
>
> So seed a token before you exercise anything else, or your first stubbed call dies on auth with a
> message that mentions neither your stub nor the token request. Then write one test that asserts this
> exception specifically, because your production error handler has to catch it separately.

Cover the **typed subclass** as well: this API documents at least one error model, so `src/Exceptions/`
carries a class for it, and a catch ladder in the wrong order (base before typed) only shows up in a test
that throws the typed one. It reaches a test the same way `ApiException` does — so if nothing above raises
on a status, nothing raises this either, and it arrives as a payload instead.

*Stub a returned failure for the status path.*
`isSuccess()`, `isError()`, `getStatusCode()` and `getResult()` are all non-final, so a
`createMock(ApiResponse::class)` can stand in:

```php
$failure = $this->createMock(ApiResponse::class);
$failure->method('isSuccess')->willReturn(false);
$failure->method('getStatusCode')->willReturn(409);
$failure->method('getResult')->willReturn($conflictErrorModel);   // or the raw decoded array — see below

$controller->method('{operation}')->willReturn($failure);
```

Every non-2xx arrives this way in a wrapper build, so assert that your gateway translated the failure.
What `getResult()` should return in the stub is decided by whether the operation's response handler calls
`mapErrorTypesInApiResponse()` — `grep -rn mapErrorTypesInApiResponse src/Controllers/` settles it, and
the presence of `ApiException.php` does not. Where it is called you get the deserialized error model
shown above. Where it is not — a common shape — nothing maps the failure body and you must stub the
decoded body instead, because a mock returning a model there hides exactly the "call to a member
function on array" bug the test exists to catch.

> **Match the decoded shape exactly, or the stub hides the bug one level down.** The SDK does not set
> `jsonOpts`, so the body is decoded to `stdClass` and only the **top level** is cast to an array. A stub
> built with `json_decode($json, true)` is associative all the way down, so `$result['details'][0]['issue']`
> passes in the test and fatals in production. Build the stub the way the SDK does —
> `(array) json_decode($json)` — leaving the nested values as objects. See *"errors arrive inside the
response"* in **php-error-handling**. (If you need a real instance rather than a double,
`ApiResponse::createFromContext($decodedBody, $result, HttpContext $context)` is the factory.)

## Observing real traffic — `httpCallback`

`httpCallback(…)` is the SDK's only introspection hook. It takes a `CoreCallback`; the SDK's own
`PaypalServerSdkLib\Http\HttpCallBack` is that class, constructed with two optional callables:

```php
use PaypalServerSdkLib\Http\HttpCallBack;

$requests = [];   // one HttpRequest per call, in order
$contexts = [];   // one HttpContext per call, in order

$client = {Client}Builder::init()
    ->httpCallback(new HttpCallBack(
        function ($request) use (&$requests): void {
            $requests[] = $request;                  // HttpRequest, before it is sent
        },
        function ($context) use (&$contexts): void {
            $contexts[] = $context;                  // HttpContext: getRequest() + getResponse()
        }
    ))
    ->build();
```

**Keep the two halves in separate variables, and keep a list of each.** The two callbacks receive
*different types*, so writing both into one variable leaves it holding whichever fired last — an
`HttpContext`, whose whole surface is `getRequest()` and `getResponse()`. Calling `getQueryUrl()` or
`getHttpMethod()` on that is a fatal error, and those are exactly the accessors you reach for next. A
list rather than a scalar matters for the same reason it does anywhere here: a call whose scheme fetches
a token fires both callbacks for the token request too, before the operation's own, so a scalar would
leave you asserting against the token exchange.

Assert off the captured objects — `getHttpMethod()`, `getQueryUrl()`, `getHeaders()`, `getParameters()`
on an `HttpRequest`; `getStatusCode()`, `getHeaders()`, `getRawBody()` on the `HttpResponse` reached
through `$contexts[$i]->getResponse()`. Select the entry you mean by URL rather than taking the last.

> **`httpCallback()` silently ignores anything that is not a `CoreCallback`** — its body is
> `if (!$httpCallback instanceof CoreCallback) { return $this; }`. A hand-rolled closure or a plain
> object is accepted by PHP and then does nothing, with no error. Use `HttpCallBack`.

This is an **observation** hook, not a stub: the request is still sent. It belongs in integration tests.

## Asserting URL resolution without a request

```php
$this->assertSame('https://expected.host/base', $client->getBaseUri());
```

Cheap, offline, and it catches the single most common misconfiguration — the wrong `Environment`
constant.

## HTTP-level fakes

Because the base URL is derived from `Environment` constants and nothing else, you can only point the
SDK at a local fake server if the API itself declares an environment (or a server parameter) whose URL
you control. Check `src/Environment.php` and `src/{Client}Builder.php`. If neither exists, an HTTP-level
fake is not reachable from configuration — use the interface or double seams above rather than editing
the SDK or patching `/etc/hosts`.

Where a suitable environment *does* exist, point at it and keep the failure fast. Read the real constant
names out of `src/Environment.php` first — most generated SDKs declare only the API's own environments,
so there is often no local one to name and this whole route is closed:

```php
$client = {Client}Builder::init()
    ->environment(Environment::{LOCAL_CONSTANT})
    ->timeout(2)                 // seconds
    ->enableRetries(false)       // a stubbed 5xx fails immediately instead of backing off
    ->build();
```

Be explicit about it — a project-wide factory may have turned retries on.

## Notes

- **Never assert on `ApiHelper::stringify` output or `__toString()`** — it is a readable dump, not a
  stable contract.
- To check what a request body will serialize to without sending anything,
  `json_encode($model)` — models implement `\JsonSerializable`.
- A success-path status code **is** available here — `getStatusCode()` on the returned `ApiResponse` — so
  assert it directly rather than reaching for `httpCallback`. Where the SDK genuinely exposes nothing you
  need, assert at the seam you own instead.

## Next

That is the last step of the workflow. If you have not read **php-configuration-resilience** yet, read it before you ship — its subject is the client's defaults, which apply whether or not you set anything.
