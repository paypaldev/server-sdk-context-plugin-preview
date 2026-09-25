---
name: 'php-error-handling'
description: 'Handle errors from the PayPal Server SDK PHP SDK. Load before your first try/catch around a call, or when building an error-translation layer. The class list won''t tell you whether a non-2xx throws at all in this SDK, where the error payload actually lives, or that an inline `catch` inside a namespaced file never matches.'
---

# Error handling for an APIMatic PHP SDK

## First: where does a failure surface?
Two independent facts about this SDK decide it, and **the first is already settled**: it returns the
`ApiResponse` wrapper — either it was built to return complete responses, or it never raises on an error
status, either of which produces the wrapper — so `src/Http/ApiResponse.php` is generated, every operation returns the wrapper, and
**an error status never raises**. Each handler chain ends `->returnApiResponse()`, which makes the runtime
return the wrapper on a failure instead of throwing.

One question is left, and the filesystem answers it — settle it before you write any `try`:

> **Does `src/Exceptions/ApiException.php` exist?** (an SDK that maps its error types into the response
> omits it)
> - **Yes** → there is something throwable, but only for a **transport** failure; a non-2xx still comes
>   back inside the wrapper.
> - **No** → nothing is throwable at all. The classes under `src/Exceptions/` are plain error *models*,
>   and a `try/catch` around a call catches nothing — not even a network failure.

That leaves two builds this SDK can be:

| `ApiException.php` | A non-2xx status | A transport failure |
| --- | --- | --- |
| present — the **wrapper build** | returns the wrapper; `getResult()` is the **raw decoded body, top level only cast to an array** | throws `ApiException` with a `getCode()` of `0` |
| absent — the **mapped build** | returns the wrapper; `getResult()` is the **deserialized error model** for any status the operation maps | nothing is raised |

Cross-check the mapped build against `src/Utils/CompatibilityConverter.php`: with no `ApiException` to
build, its `createApiException()` body is `return null;`.

Nothing in this SDK converts a `404` into `null`: it surfaces exactly like
any other non-2xx, by the mechanism above, and no `nullOn404()` appears anywhere in the controllers.

## The wrapper builds — errors arrive inside the response

In the wrapper and mapped builds every operation returns `ApiResponse`, a non-2xx never raises, and the
failure comes back on `getResult()`. Branch on the response:

```php
$apiResponse = $controller->{operation}(/* … */);

if (!$apiResponse->isSuccess()) {
    $status = $apiResponse->getStatusCode();
    $error  = $apiResponse->getResult();       // the deserialized error model, or the raw body — see below
    $body   = $apiResponse->getBody();         // the original body
}
```

`ApiResponse` also exposes `isError()`, `getHeaders()` and `getRequest()`.

**What `getResult()` holds separates the two.** With `src/Exceptions/ApiException.php` **absent** (the
mapped build) the handler chain also calls `mapErrorTypesInApiResponse()`, so `getResult()` is the
deserialized error model; the classes under `src/Exceptions/` *are* those models, emitted with no `extends`
clause, so `catch ({Typed}Exception $e)` can never match one even though the name ends in `Exception`. With
`ApiException.php` **present** (the wrapper build) nothing maps the failure body, so `getResult()` is the
**raw decoded body** and `->getMessage()` on it is a fatal "call to a member function on array"; there,
keep a `try/catch` for `ApiException` around the call as well, because a transport failure still raises.
Confirm on the `class ...` line before writing a catch block.

> ### ⚠ The decoded body is not an array all the way down
>
> **Only its top level is cast.** The runtime decodes with no `jsonOpts`, so the body arrives as a
> `stdClass` tree, and the accessor behind `getResult()` applies a single `(array)` — which converts one
> level and does not walk. Read the runtime's own accessor rather than assuming; nothing in the generated
> code shows this.
>
> The result is a hybrid shaped so the wrong reading appears to work. The top level is an array, its
> scalar members are strings, and a JSON **array** member decodes to a PHP array even in object mode — so
> `$result['message']` and `count($result['items'])` both succeed. An **object nested inside** that array
> is still a `stdClass`, so the next step down is where it fails:
>
> ```php
> $result['message']              // string  — array access works
> $result['items'][0]['code']     // Error: Cannot use object of type stdClass as array
> $result['items'][0]->code       // works — the nested value is an object
> ```
>
> Read the top level with array syntax and anything nested with `->`, or decode the raw body yourself
> with `json_decode($apiResponse->getBody(), true)` when you want one associative structure throughout.
> The usual defences do not help: `is_array($result)` is `true`, and `array_key_exists()` against a
> nested object is itself a `TypeError`. This lives in the runtime package, so it is true of every
> generated PHP SDK on a wrapper build, not a property of this one.

## `ApiException` — the throwing paths

Generated wherever `src/Exceptions/ApiException.php` exists — the wrapper build, not the mapped one. There
it is raised for a **transport failure only**: every operation returns the wrapper, so a non-2xx never
reaches a `catch` block at all.

`PaypalServerSdkLib\Exceptions\ApiException` extends `\Exception`, and its docblock is exact: *"Thrown
when there is a network error or HTTP response status code is not okay."* Both cases, one type — but only
the network half ever fires here.

| Member | Meaning |
| --- | --- |
| `getCode(): int` | the HTTP status — **or `0` when no response was received at all** |
| `getMessage(): string` | the failure reason (for a documented error, the description from the API spec) |
| `hasResponse(): bool` | whether a response was received |
| `getHttpResponse(): ?HttpResponse` | `getStatusCode(): int`, `getHeaders(): array`, `getRawBody(): string` |
| `getHttpRequest(): HttpRequest` | `getHttpMethod()`, `getQueryUrl()`, `getHeaders()`, `getParameters()` |

**The `getCode() === 0` trap.** A DNS failure, a refused connection, a TLS error or a timeout produces an
`ApiException` with code `0` and no response. Code that branches on `$e->getCode() >= 500` therefore
treats a network outage as a client error. Test `hasResponse()` first:

```php
use PaypalServerSdkLib\Exceptions\ApiException;

try {
    $result = $controller->{operation}(/* … */);
} catch (ApiException $e) {
    if (!$e->hasResponse()) {
        // transport failure: no status, no body
        throw new MyTransportError($e->getMessage(), 0, $e);
    }
    $status = $e->getCode();
    $body   = $e->getHttpResponse()->getRawBody();
}
```

### Typed exceptions — catch them first
This API documents at least one error model, so `src/Exceptions/` carries a class per documented error
response.

**But none of them is ever thrown in this SDK**, so the heading is a warning rather than an instruction: a
non-2xx comes back inside the wrapper, and the `throwErrorOn` entry naming a typed class picks a mapping
target, not a throw. `catch ({Typed}Exception $e)` is dead code here — and in the mapped build it could not
match anyway, because the class is emitted with no `extends` clause even though its name ends in
`Exception`. What you want from these classes is their **payload getters**, for reading whatever
`getResult()` hands you:

```php
$apiResponse = $controller->{operation}(/* … */);

if (!$apiResponse->isSuccess()) {
    $error = $apiResponse->getResult();      // a {Typed}Exception instance ONLY in the mapped build;
                                             // otherwise the raw decoded array — see above
    error_log(is_object($error) ? $error->get{Field}() : json_encode($error));
}
```

> **The `Property` suffix appears only on error classes that extend `ApiException`**, where the payload
> name would collide with `\Exception`'s own `getCode()` / `getMessage()` — there a JSON `code` field
> becomes `getCodeProperty()`. When the error class is a plain model (no `extends`), the getters keep
> their JSON names: `getCode()`, `getMessage()`. **The class in `src/Exceptions/` is authoritative.**

## Which errors does an operation map?
Two places tell you, both authoritative:

- **`doc/controllers/{group}.md`** — every operation has an *Errors* table mapping HTTP status to the
  generated error class.
- **The operation body in the controller directory** — `src/Controllers/` by default, but named per SDK
  after the controller postfix or its namespace (`src/Apis/` when the postfix is `Api`). Find it
  with a grep for `throwErrorOn` under `src/`: the chain reads
  `->throwErrorOn('<status>', ErrorType::init(…, {Typed}Exception::class))`, and a `throwErrorOn('0', …)`
  entry is the catch-all default for otherwise-unmapped statuses. **The chain is the status→class map, not
  proof of a throw** — in a wrapper build it never raises.

An operation with no mapped errors surfaces the base `ApiException` only in the bare shape, or an
`ApiResponse` whose `getResult()` is undeserialized in the wrapper shapes. This is common.

## Failures no build covers

These are raised before or during serialization, in **every** build, and never extend `ApiException`:

- **Enum and union validation.** Passing a value outside a generated enum, or a value that doesn't match
  a `@mapsBy` union template, throws a plain `\Exception` — for enums with the message
  `"<value> is invalid for {EnumClass}."`. See **php-models**.
- **Missing or unobtainable OAuth authorization.** *Any* OAuth grant throws `\InvalidArgumentException`
  from the auth manager's `validate()` before the request is sent — "Client is not authorized. An OAuth
  token is needed to make API calls." or "OAuth token is expired…". On the automatic grants a failed
  token fetch is swallowed and re-surfaces as that same "Client is not authorized" message, so read it as
  "could not obtain a token", not just "no token was configured". See **php-authentication**.
- **Additional-property key collisions** throw `\InvalidArgumentException`.

Catch these separately, or let them propagate — they signal a bug in the calling code or the credentials,
not a server condition.

## Notes

- **Retries have already happened** by the time an error reaches you, and only for the HTTP methods in
  `ConfigurationDefaults::HTTP_METHODS_TO_RETRY` — and only if retries were enabled at all. A `POST` that
  surfaces a `503` most likely never got a second attempt. See **php-configuration-resilience**.
- `getRawBody()` is a **string**, not decoded JSON. Use `json_decode(…, true)` when you need fields the
  error class doesn't expose, and guard against a non-JSON body (HTML error pages from a proxy are
  common).
- Rate-limit information lives in response headers — `$apiResponse->getHeaders()` on a returned wrapper,
  `$e->getHttpResponse()->getHeaders()` on a caught `ApiException`.
- `echo $e;` on an `ApiException` gives a readable dump (it implements `__toString()` via
  `ApiHelper::stringify`), which is fine for a log line but not something to parse.
- Do not swallow the failure to return `null` — the caller then cannot distinguish "not found" from "the
  API is down". Translate into your own domain error and keep the original as the `$previous`.

## Next

- Step 6, retries, timeouts, proxy, logging and the base URL → **php-configuration-resilience**
- Step 7, stubbing the SDK → **php-testing**

Retry and timeout behaviour is named above only where an error surfaces it. The **defaults** — what this client does when you configure nothing — belong to **php-configuration-resilience**, and reading them here is not a substitute for reading them there.
