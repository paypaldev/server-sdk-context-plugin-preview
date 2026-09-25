---
name: 'java-calling-endpoints'
description: 'Call operations on the PayPal Server SDK Java SDK. Load before the first call, and when building a request body or reading a response. The signature won''t tell you the controller package and class suffix are both per-SDK, that every operation has a blocking and an async form, or what the return type wraps.'
---

# Calling endpoints on an APIMatic Java SDK

> **`doc/` is in the SDK's repository, not in the installed package.** Every pointer to
> `doc/controllers/` or `doc/client.md` below assumes you have the repository checked out. Where you
> only have the installed artifact — the common case — the equivalent source is the operation's own signature, read from the
> sources jar where one is published or from your IDE's view of the artifact. It is authoritative in a
> way the docs are not.

Operations are methods on a **controller you get from the client**, not on the client itself:

```java
import {rootPackage}.{Api}Client;
import {rootPackage}.{controllerPackage}.{Controller};
import {rootPackage}.exceptions.ApiException;

{Api}Client client = new {Api}Client.Builder()./* ... */.build();
{Controller} controller = client.get{Controller}();

{ReturnType} result = controller.{operation}(/* ... */);
```

Controllers live in `<root>/<controllerPackage>/`, one class per API group. **The package name and the
class suffix are both named per SDK** — `controllers`/`Controller` by default, and either can be
something else. The suffix may also be absent entirely, so the accessor may be
`getPetsController()`, `getPetsApi()` or `getPets()`. **Take the package from the `import` block at the top of
`{Api}Client.java` and the accessor from its `get...()` methods**; `doc/client.md` lists them all.

**This SDK ships no controller interfaces**, so each group emits a single `final` class and
no interface: `{Controller}.java` *is* the implementation, and it is the file to open for an exact
signature. The client implements `Configuration` directly.

## Two method shapes: blocking and `...Async()`

The **blocking** form is always generated:

```java
public {ReturnType} {operation}({params}) throws ApiException, IOException
```

When the SDK was generated in asynchronous mode, each operation also gets a twin:

```java
public CompletableFuture<{ReturnType}> {operation}Async({params})
```

- The blocking form declares **two checked exceptions** — `ApiException` for any non-2xx response and
  `IOException` for transport failures. XML operations add `JAXBException`. You must handle or declare
  both; see **java-error-handling**.
- The async form declares **no** checked exceptions: it wraps everything into a `CompletionException`,
  so the real error is at `exception.getCause()`.
- Both forms exist side by side and share one request builder, so picking either is purely a style
  choice. The generated `doc/controllers/*.md` shows only one of them — open the `.java` file to see
  both.

```java
// blocking
try {
    {ReturnType} result = controller.{operation}(/* ... */);
} catch (ApiException e) {
    // non-2xx
} catch (IOException e) {
    // network/transport
}

// async
controller.{operation}Async(/* ... */)
        .thenAccept(result -> { /* ... */ })
        .exceptionally(exception -> {
            Throwable cause = exception.getCause();   // the real error
            return null;
        });
```

## Parameters — positional, or one collapsed `Input` object

Which form an operation uses is fixed when the SDK is generated — for the whole SDK, or for that one
endpoint on its own — and only ever applies to an operation with **more than one non-constant
parameter**. Neither form is the norm — in a collapsing build nearly every multi-parameter
operation takes an `{Operation}Input`, and in a build without it none does. **Read the signature in the
controller class the client's `get...()` accessor returns** before you write the call.

That count is of **declared** parameters, not of required ones: the optional ones count towards it too,
so an operation taking one required path parameter plus a few optional headers or filters is a
multi-parameter operation and does collapse on such a build. Only an operation declaring exactly one
parameter, with no optional ones behind it, stays bare.

An operation that was not collapsed — and every genuinely one-parameter operation, whatever the setting —
takes its parameters **positionally**, in the order the method declares them:

```java
{ReturnType} result = controller.{operation}(petId, status);
```

Optional parameters are still positional — pass `null` for the ones you want to omit. Boxed types
(`Integer`, `Long`, `Boolean`, `Double`) are used precisely so that `null` is expressible.

A collapsed operation instead takes a **single `{Operation}Input` object** bundling every parameter. The
parameter is literally named `input`:

```java
public {ReturnType} {operation}(final {Operation}Input input) throws ApiException, IOException
```

`{Operation}Input` is an ordinary generated model in `<root>/models/`, built the same way as any other —
required values in the `Builder` constructor, optional ones as fluent setters (see **java-models**):

```java
{ReturnType} result = controller.{operation}(new {Operation}Input.Builder(requiredA, requiredB)
        .optionalC(value)
        .build());
```

Two operations in the same controller can differ: with collapsing on, one that takes a single parameter
still takes it bare, so check each signature rather than generalising from the first.

Some operations append one or both of `final Map<String, Object> queryParameters` and
`final Map<String, Object> fieldParameters` after the regular parameters. Those are escape hatches for
additional, unmodelled query or form values.

> **Pass an empty map, not `null`.** The generated body iterates the map without a null check, so a
> `null` throws `NullPointerException` before the request is built. `Collections.emptyMap()` is the
> no-extras call.

There is **no per-call options argument** — no request options, no per-call timeout, no cancellation
token. Timeouts and retries are client-level; see **java-configuration-resilience**.

## Building request bodies

A body parameter is a generated model. Required properties go in the model `Builder`'s constructor,
optional ones are fluent setters:

```java
import {rootPackage}.models.{Model};

{Model} body = new {Model}.Builder(requiredProp)
        .optionalProp(value)
        .build();

{ReturnType} result = controller.{operation}(body);
```

Models also expose a public all-args constructor, so `new {Model}(a, b, c)` works too — but the
positional constructor breaks the moment a regeneration adds a field, while the `Builder` does not.
**This SDK's models are mutable**, so every field also has a plain setter and the
model has a no-arg constructor.

For enums, unions, dates, collections and nullable-optional
fields, load **java-models**.

## Reading the response


> **Every non-paginated return type in this SDK is wrapped in `ApiResponse<T>`**, from
> `<root>/http/response/`. That applies to the whole SDK at once.

```java
ApiResponse<{Model}> response = controller.{operation}(petId);
int status          = response.getStatusCode();
Headers headers     = response.getHeaders();
{Model} value       = response.getResult();
```

The payload is on **`getResult()`** — the extra hop is what people forget. An operation with **no
response body** is wrapped too: it returns `ApiResponse<Void>` in the blocking form and
`CompletableFuture<ApiResponse<Void>>` in the async one, so even a `DELETE` hands you a status code.

A binary or file response comes back as `ApiResponse<InputStream>` — read and close the stream
yourself.

## File uploads

File parameters are `FileWrapper` from `<root>/utilities/`:

```java
import {rootPackage}.utilities.FileWrapper;

FileWrapper upload = new FileWrapper(new File("./photo.png"), "image/png");
{ReturnType} result = controller.{operation}(upload);
```

The one-argument `new FileWrapper(file)` constructor also exists when the content type is implied.

## Pagination — none in this SDK

**This API declares no paginated operation.** The SDK ships no `<root>/utilities/pagination/` package,
no `PagedIterable`/`PagedFlux`/`PagedSupplier` types, and no operation returns one. A list endpoint here
returns the ordinary shape described above: drive its own page/offset/cursor parameters yourself and
stop on whatever the API uses to signal the end.

## Finding the right method in the SDK source

- Start with `doc/controllers/*.md` — it names the controller class, the accessor, every operation, its
  parameters, its response type and its documented errors, with a copy-pasteable sample.
- Then open the controller source itself — `{Controller}.java` in whatever package the client imports it
  from — for the exact signature: the parameter list (positional or `Input`), the `throws` clause, and
  the return type.
- Request/response/enum types are under `<root>/models/`; typed exceptions under `<root>/exceptions/`.

## Next

- Enums, unions, dates, nullable-optional fields → **java-models**
- Errors and status codes → **java-error-handling**
- Retries, timeouts, logging → **java-configuration-resilience**
