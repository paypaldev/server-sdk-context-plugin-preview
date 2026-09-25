---
name: 'java-error-handling'
description: 'Handle errors from the PayPal Server SDK Java SDK. Load before your first try/catch around a call, or when building an error-translation layer. The throws clause won''t tell you the status code accessor is `getResponseCode()`, which errors carry a typed subclass and which do not, or that the async form buries everything in a `CompletionException`.'
---

# Error handling for an APIMatic Java SDK

Every blocking operation declares **two** checked exceptions:

```java
public {ReturnType} {operation}({params}) throws ApiException, IOException
```

- **`ApiException`** — the server answered, but not with a success status. Lives in
  `<root>/exceptions/` and extends the runtime's `CoreApiException`.
- **`IOException`** — the request never completed: connection refused, DNS failure, TLS error, socket
  timeout. It is *not* an `ApiException`, so a catch block that only handles `ApiException` will not
  compile without also handling it.
- **`AuthValidationException`** (`io.apimatic.core.exceptions`) — **unchecked**, so it appears in no
  `throws` clause. Thrown before the request is sent when a required credential is missing or an OAuth
  token cannot be obtained. It extends `RuntimeException`, so neither `catch (ApiException)` nor
  `catch (IOException)` covers it: add an explicit `catch (AuthValidationException e)`, or let it escape
  as the configuration bug it is. See the *Notes* below.

XML-based operations add `JAXBException`.

## Two shapes of `ApiException`

- **Typed subclass (Case A)** — for an error response the spec gives a **modelled body**, the generator
  emits a per-error class under `<root>/exceptions/` extending `ApiException`, with Jackson-annotated
  fields and public getters for the error payload. At least one error in this API has such a body, so
  those classes exist. `doc/controllers/{group}.md` has an **Errors** table naming which
  class each *explicitly-listed* status maps to — but the table is emitted for **every** documented error
  and its *Exception Class* column falls back to the base `ApiException` name where no model exists, so a
  row there is not proof that a typed class was generated. The `.localErrorCase(...)` calls in the
  operation's own response handler (plus the base controller's `.globalErrorCase(GLOBAL_ERROR_CASES)`
  default) are the authority. Treat the table's `Default` row as advisory only too
  — the runtime ignores a local `DEFAULT` error case and falls back to the plain
  `ApiException("HTTP Response Not OK")` registered in the base controller, so never make the `Default`
  row's class your only catch for unexpected statuses.
- **Base `ApiException` (Case B)** — every SDK registers a default case that produces a plain
  `ApiException` (reason `"HTTP Response Not OK"`) for any non-2xx not matched by an explicitly-listed
  status code (or a `4XX`/`5XX` key) — including statuses the spec covered only through a `default`
  response. This is the common path.

Both can come out of the same call, so a typed catch always needs an `ApiException` fallback after it.

## Catch it

```java
import {rootPackage}.exceptions.ApiException;
import {rootPackage}.exceptions.{Operation}Exception;
import java.io.IOException;

try {
    {ReturnType} result = controller.{operation}(/* ... */);
    // use result
} catch ({Operation}Exception e) {                 // Case A — MUST come first
    // typed payload accessors, e.g.:
    System.err.println(e.getResponseCode() + " " + e.getCode());
} catch (ApiException e) {                          // Case B — any other non-2xx
    System.err.println("HTTP " + e.getResponseCode());
} catch (IOException e) {                           // transport failure
    System.err.println("transport: " + e.getMessage());
}
```

Java resolves catch blocks top-down, so **a `catch (ApiException e)` placed before the typed subclass is
a compile error** ("already caught"). Order most-specific first.

## Reading the error

`ApiException` gives you:

| Member | Returns | Notes |
| --- | --- | --- |
| `getResponseCode()` | `int` | the HTTP status code. **Not** `getStatusCode()` — that one is on `HttpResponse` |
| `getHttpContext()` | `HttpContext` | the request/response pair, or `null` when the exception was raised before a response existed |
| `getMessage()` | `String` | the *reason* string the SDK set: the error description from the API spec for a documented error, `"HTTP Response Not OK"` otherwise — **not** the API's `message` field |

Through `getHttpContext()` you reach the wire detail:

```java
HttpContext context = e.getHttpContext();
if (context != null) {
    HttpRequest request   = context.getRequest();
    HttpResponse response = context.getResponse();

    int status      = response.getStatusCode();
    Headers headers = response.getHeaders();     // rate-limit headers, request ids, ...
    String body     = response.getBody();        // also getRawBodyString() / getRawBody()
    String url      = request.getQueryUrl();
}
```

Always null-check `getHttpContext()` before dereferencing it.

## The `getMessageField()` trap

A typed exception's fields come from the error schema, but a Java exception already owns some of those
names. When a field's accessor would collide with `getMessage`, `getResponseCode`, `getCause`,
`getSuppressed`, `getStackTrace`, `getLocalizedMessage` or `getClass`, the property carries a `Field`
suffix:

```java
// JSON: { "code": 400, "message": "Bad request" }
catch ({Operation}Exception e) {
    int code       = e.getCode();
    String apiText = e.getMessageField();   // the API's "message"
    String reason  = e.getMessage();        // the SDK's reason string — NOT the API's message
}
```

Reading `getMessage()` and expecting the server's text is the single most common mistake here. Open the
exception class under `<root>/exceptions/` and use the accessors it actually declares.

## Async operations

The `...Async()` form declares no checked exceptions — it wraps everything in a
`java.util.concurrent.CompletionException`. The real error is one level down:

```java
controller.{operation}Async(/* ... */)
        .thenAccept(result -> { /* ... */ })
        .exceptionally(exception -> {
            Throwable cause = exception.getCause();

            if (cause instanceof {Operation}Exception) {
                {Operation}Exception typed = ({Operation}Exception) cause;
                System.err.println(typed.getResponseCode());
            } else if (cause instanceof ApiException) {
                System.err.println(((ApiException) cause).getResponseCode());
            } else {
                exception.printStackTrace();
            }
            return null;
        });
```

Testing `exception instanceof ApiException` on the `CompletionException` itself always fails — go through
`getCause()`. The same applies to `join()`/`get()` on the future.

## Notes

- **Auth is validated before the request is sent**, and the failure is an unchecked
  `AuthValidationException`, not an `ApiException`. For a client-credentials grant the SDK first tries to
  fetch a token; if *that* call fails (bad client id/secret, unreachable token server) the
  `ApiException`/`IOException` is swallowed inside `{Scheme}Manager.getTokenFromProvider()` and you still
  get the generic *"An OAuth token is needed to make API calls."* message — so the status code you want
  was received and discarded. To see it, call `fetchToken()` yourself once at startup on the credentials
  getter `{Api}Client.java` actually declares — `get{Scheme}Credentials()` in a multi-scheme SDK, the
  suffix-less `getClientCredentialsAuth()` when that grant is the API's only scheme — and catch it there.
  Fix the credentials rather than the catch block; see **java-authentication**.
- **Retries happen before the exception reaches you** — see **java-configuration-resilience** for what
  this SDK retries, and do not assume a `POST` was.
- **A 404 throws like any other non-2xx.** Every operation's request builder in this SDK carries an
  explicit `.nullify404(false)` — so there is no silent `null` return to guard
  against and the `catch` ladder above covers 404 too.
- **Do not leak SDK exceptions across your own API boundary.** Translate them in one place into your own
  domain errors, keying off `getResponseCode()` and the typed payload, so callers do not depend on the
  SDK's package names.

## Next

- Step 6, retries, timeouts, transport, logging and the base URL → **java-configuration-resilience**
- Step 7, stubbing the SDK → **java-testing**

Retry and timeout behaviour is named above only where an error surfaces it. The **defaults** — what this client does when you configure nothing — belong to **java-configuration-resilience**, and reading them here is not a substitute for reading them there.
