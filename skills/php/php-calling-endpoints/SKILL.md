---
name: 'php-calling-endpoints'
description: 'Call operations on the PayPal Server SDK PHP SDK. Load before the first call, and when building a request body or reading a response. The signature won''t tell you the controller folder and class suffix are per-SDK, whether the operation returns a wrapper or the bare value, or that a `404` can arrive as `null`.'
---

# Calling endpoints on an APIMatic PHP SDK

> **`doc/` is in the SDK's repository, not in the installed package.** Every pointer to
> `doc/controllers/` or `doc/client.md` below assumes you have the repository checked out. Where you
> only have the installed artifact — the common case — the equivalent source is the `->extract('…')` calls in the generated method body under `src/Controllers/`, and it is
> authoritative in a way the docs are not.

Operations are methods on a **controller you get from the client** — you never construct one:

```php
$controller = $client->get{Group}{Postfix}();   // e.g. getShipmentsController() — or getShipmentsApi()
                                                // whichever suffix this SDK uses
$result = $controller->{operation}(/* … */);
```

The client memoizes each controller, so calling the accessor repeatedly is free. Controllers live in
`src/{Postfix}s/` — the directory and namespace are the *pluralised* class postfix, which is named per
SDK, so `src/Controllers/` when the postfix is `Controller`,
`src/Apis/` when it is `Api` — and each extends a shared `Base{Postfix}` (`BaseController`, `BaseApi`, …).
Find the folder with `ls src/`: it is the one that is not `Models`/`Http`/`Utils`/`Authentication`/
`Exceptions`/`Logging`/`Proxy`. **Read the accessor names from `src/{Client}.php`.** Operation names
follow no fixed verb/resource pattern; take the real name from the source or from `doc/controllers/`.

## Two parameter conventions — check which one the method uses

Neither form is the norm, and neither follows from the parameter count alone. An operation with **more
than one** non-constant parameter is collapsed into a single `array $options` in SDKs built to collapse
them, and in any SDK where that one endpoint collects its parameters, and stays positional otherwise; an operation declaring **one** parameter is positional either way. Both conditions have to hold,
so a single SDK routinely ships both shapes side by side — the collapsed form for its multi-parameter
operations and the positional form for every one-parameter operation in the same controller. Count the
**declared** parameters, optional ones included, not the required ones: one required id followed by a few
optional filters is a multi-parameter operation. So **read a signature in the controller folder before you
write the first call** — do not carry a call shape over from another operation or another SDK.

### 1. Positional parameters

```php
public function {operation}(
    string $requiredParam,
    ?int $optionalParam = null,
    ?{Model} $body = null
): {ReturnType}
```

- Required parameters come first and are typed non-nullably.
- Optional parameters are `?T`. Most default to `null`, but a parameter with a default in the spec is
  generated with **that** default (`?int $limit = 25`, `?bool $includeExpired = false`). **Copy the
  default out of the signature before skipping a parameter** — passing `null` past a non-null default
  omits it from the request instead of sending the default, silently changing the call. Prefer PHP 8
  named arguments, or re-supply the printed default, over padding with `null` (these SDKs still support
  PHP 7.2, so named arguments depend on your project's minimum version).
- **An enum parameter is typed `string` or `int` in the signature, not as the enum class.** The enum
  class only appears in the docblock and in `doc/controllers/*.md` (rendered as `?string(EnumName)`).
  Pass a constant off the generated class: `{EnumClass}::{CONSTANT}`. A closed enum's
  value is validated when the request is serialized, so an unlisted string fails there rather than at the
  call site.

### 2. A single `array $options` (the "collect parameters" mode)

In a collapsing build every parameter of such an operation — headers, query, path **and the request
body** — becomes one key of one associative array:

```php
public function {operation}(array $options): {ReturnType}
```

```php
$collect = [
    'body'        => $body,                        // the request model, when the operation has one
    'xSomeHeader' => 'value',
    'resultType'  => {EnumClass}::{CONSTANT},
    'limit'       => 50,
];

$result = $controller->{operation}($collect);
```

The keys are the **camelCased parameter names**, and nothing checks them — **including the required
ones**. Whether a key is required is expressed by `->required()` on the parameter, and plenty of SDKs
never emit it: `grep -rc 'required()' src/Controllers/` settles it for yours in one command. Where
it is absent, a misspelled `'bdy'` or `'ids'` is dropped exactly as silently as a misspelled optional
key, and the request goes out malformed rather than raising anything client-side. **A typo becomes a
provider error, not an exception** — so treat the key list as something to copy, never to type.

For the same reason, do not go looking for a default in the signature: a spec default is applied by
`->extract($key, $default)` when the key is *absent*, not written into the method's parameter list.

Get the exact key list from the *Parameters* table in `doc/controllers/{group}.md` where you have the
repository, or from the `->extract('…')` calls in the generated method body, which you always have.

### A trailing `?array $fieldParameters = null`

An operation whose endpoint accepts arbitrary extra **form** fields ends with
`?array $fieldParameters = null` — token endpoints generated from an OAuth scheme are the usual case.
Pass a plain `['key' => 'value']` map; the generated body wires it through
`AdditionalFormParams::init(...)`, so the entries go into the **request body**, not the query string.
It stays the last argument in a collapsing build too:
`{operation}(array $options, ?array $fieldParameters = null)`.

## Building request models

A request body is a generated model class. Required fields are constructor arguments on its **builder**;
optional fields are fluent setters:

```php
use PaypalServerSdkLib\Models\Builders\{Model}Builder;

$body = {Model}Builder::init(
    /* required fields, positionally, in the builder's own order */
)
    ->{optionalField}($value)
    ->build();

$result = $controller->{operation}($body);
```

`{Model}Builder::init(...)`'s parameter list *is* the required-field list. Anything not in it is
optional. See **php-models** for enums, unions, dates, collections and the optional-vs-null distinction.

`new {Model}(...)` also works and takes the same required arguments, but then you set optionals through
`set{Field}(...)` on the instance; the builder is what the generated docs use.

**File uploads** take a `FileWrapper` — but `src/Utils/FileWrapper.php` is generated **only** when some
operation declares a file parameter, and most APIs declare none. `ls src/Utils/` before you import it: a
`use` of a class that was never generated is a fatal PSR-4 autoload failure, not a warning.

```php
use PaypalServerSdkLib\Utils\FileWrapper;

$controller->{operation}(FileWrapper::createFromPath('/path/to/file.png'));
```

## Reading the response
**This SDK returns `ApiResponse`, on every operation.** `src/Http/ApiResponse.php` is generated, nothing here hands back the bare deserialized value, and the
generated methods carry no `@throws ApiException` annotation because in this shape an error status is not
raised at all. Confirm it on any method in the controller directory: the return type reads `ApiResponse`.

The wrapper exposes `getStatusCode(): ?int`, `getHeaders(): ?array`, `getResult()`, `isSuccess()`,
`isError()`, `getBody()` (the raw body) and `getRequest()`.

**An error *status* does not throw.** Every operation's handler chain ends `->returnApiResponse()`, and the
runtime returns the wrapper on a failure rather than raising. So `isSuccess()` is the check that matters,
and that is why the generated `doc/controllers/*.md` examples branch on it with no `try/catch`. What still
raises is a **transport** failure — DNS, refused connection, TLS, timeout — as `ApiException` with a
`getCode()` of `0`, wherever `src/Exceptions/ApiException.php` exists. See **php-error-handling**.

**On a failure, `getResult()` is typed only in an SDK that maps error types into the
wrapper** — one whose handler chains also call
`mapErrorTypesInApiResponse()`. Without that call, and it is commonly absent, a failure leaves
`getResult()` holding the **raw decoded body**, not an error model — so
`->getMessage()` on it is a fatal "call to a member function on array". That call and
`src/Exceptions/ApiException.php` are two faces of one decision, so either check answers it:
`grep -rl mapErrorTypesInApiResponse src/` finds it on **every** operation when `ApiException.php` is
absent and on **none** when it exists. In the no-mapping build, **only the top level of `getResult()` is an
array** — nested objects stay `stdClass`, so `$result['items'][0]['code']` is a fatal one level below where
array syntax last worked. Read the top level with `[...]` and anything nested with `->`, or take
`getBody()` and decode it yourself with `json_decode($body, true)` for one associative structure
throughout. Why, and the shape it produces, is in **php-error-handling**.

### Worked example — a list call

No error status throws, so the status check *is* your handling for HTTP failures. Keep a `try/catch`
around it only for transport failures, and only where `src/Exceptions/ApiException.php` exists:

```php
$apiResponse = $controller->{operation}({EnumClass}::{CONSTANT}, 50);

if (!$apiResponse->isSuccess()) {         // every non-2xx lands here, mapped or not
    $error = $apiResponse->getResult();   // a mapped error model ONLY if this SDK maps error
                                          // types into the wrapper — otherwise a raw array
    // translate into your own error — see php-error-handling
    return;
}
foreach ($apiResponse->getResult() as $item) {
    echo $item->get{Field}(), PHP_EOL;
}
```

> **`ApiException` is not present in every SDK** — check whether `src/Exceptions/ApiException.php`
> exists. If it does not, this SDK maps errors into the response: `getResult()` on a failure is the
> deserialized error model, the classes under `src/Exceptions/` are those *models* rather than throwables,
> and a `try/catch` around the call catches nothing at all — not even a transport failure.

Wherever you catch `ApiException`, **import the class**: `use PaypalServerSdkLib\Exceptions\ApiException;`
at the top of the file. Written inline as `catch (PaypalServerSdkLib\Exceptions\ApiException $e)` inside a
namespaced file the name resolves **relative to the current namespace**, so the catch never matches.
Import it, or root-anchor it as `\PaypalServerSdkLib\Exceptions\ApiException`.

**The status→class map lives in the handler chain**, as `->throwErrorOn(...)` entries on each operation —
see **php-error-handling** for whether they raise.

> In this SDK no operation converts a `404` into `null` — a `404` surfaces
> exactly like any other non-2xx (see **php-error-handling**), and no `nullOn404()` appears anywhere in
> the controllers. Where a return type *is* nullable, that is the API's own optional response model.

## Paging a list endpoint
**This API declares no paginated operation**, and PHP SDKs generate no pagination support in any case — no
iterator, no page wrapper, no `paginate()`. A list endpoint here is an ordinary operation: if it takes
`page`/`offset`/`cursor`/`limit` parameters, advance them yourself. See **php-configuration-resilience**
for the loop shape.

## Finding the right method in the SDK source

- `doc/controllers/{group}.md` — fastest route: the operation list, a parameters table (with the
  `$options` key names and any enum type), the response type, and a runnable example. Start here.
- `src/{Postfix}s/{Group}{Postfix}.php` (e.g. `src/Controllers/ShipmentsController.php`, or
  `src/Apis/ShipmentsApi.php` in a renamed build) — the exact signature, plus
  the request-builder body showing which parameter maps to a header, a query parameter, a path segment or
  the form/body.
- `src/{Client}.php` — the `get{Group}{Postfix}()` accessor names.

## Next

- Request/response field shapes → **php-models**
- Errors and status codes → **php-error-handling**
- Retries, timeouts, manual paging, logging → **php-configuration-resilience**
