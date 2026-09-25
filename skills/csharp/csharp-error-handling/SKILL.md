---
name: 'csharp-error-handling'
description: 'Handle errors from the PayPal Server SDK C# SDK. Load before your first try/catch around a call, or when building an error-translation layer. The thrown type won''t tell you the status code is on `ResponseCode`, which failures never arrive as an `ApiException` at all, or that a Polly timeout does not derive from `OperationCanceledException`.'
---

# Error handling for an APIMatic C# SDK

Every error the API reports arrives as **`PaypalServerSdk.Standard.Exceptions.ApiException`** or a subclass
of it, so one `catch (ApiException)` catches every *thrown* failure.
`Exceptions/ApiException.cs` adds only a constructor and `ToString`; everything else comes from
`CoreApiException<HttpRequest, HttpResponse, HttpContext>`, which extends `System.Exception` — so
**the status code is `ResponseCode`**, not `StatusCode`.


## Which exception does an operation throw?

Each operation declares its own error map, in the `ResponseHandler` block of its body:

```csharp
.ResponseHandler(responseHandler => responseHandler
    .ErrorCase("404", CreateErrorCase("<the spec's description>",
        (errorReason, context) => new {Typed}Exception(errorReason, context)))
    .ErrorCase("0", CreateErrorCase("<the default-error description>",
        (errorReason, context) => new {Default}Exception(errorReason, context))))
```

- Each documented status gets a **typed subclass** in `Exceptions/`, carrying the error schema's
  fields as `[JsonProperty]` auto-properties.
- **`"0"` is the catch-all** for every other unsuccessful status; read which class it names, because
  an undocumented status does **not** imply the base `ApiException`.
- With **no `"0"` case**, an unmapped status falls through to a plain `ApiException` carrying no payload,
  whose `Message` *starts* `HTTP Response Not OK`. Where a `"0"` case exists its description can itself be
  a template that appends the status and the raw body — `HTTP Response Not OK. Status code: 500. Response:
  '{"message":"boom"}'.` — so **match with `StartsWith`, never `==`**, and branch on `ResponseCode`
  instead wherever you can.

**There is a second, independent place errors can be registered, and it is additive — not an
alternative.** The controller base class may hold a static `GlobalErrors` dictionary that it passes into
**every** `CreateApiCall<T>` via `globalErrors:`. Where it exists, those cases apply to every operation in
the controller *including* operations that also have their own `ResponseHandler`, so an operation's own
map is not the whole story.

To settle it for an SDK, open the base class in the controller folder (its name varies per SDK)
and look for a `GlobalErrors` member — do **not** rely on grepping the folder for `ErrorCase`, because the
base class declares a `CreateErrorCase` helper in every generated SDK and so always matches. Three
distinct shapes exist:

- **A `GlobalErrors` map plus per-operation handlers** — union the two; the global cases still fire.
- **A `GlobalErrors` map and no per-operation handlers** — every operation shares that one map.
- **No `GlobalErrors` member at all** — then an operation with no `ResponseHandler` maps nothing, and every
  unsuccessful status is a bare `ApiException`. Do not write a typed catch for it; it can never fire.

`doc/controllers/{group}.md` *may* repeat the map as a per-operation **Errors** table, and
`doc/models/{typed-exception}.md` *may* hold the class's fields and a worked catch snippet — but neither is
guaranteed. Some SDKs emit no `## Errors` section at all, in which case the registrations above are the
only authority. The `.cs` always wins over the `doc/`.

## Catch the exception

Every typed exception extends `ApiException`, but **not always directly** — an error schema that
extends another emits an exception class extending that other one, so read the class declaration
before ordering a ladder. `catch (ApiException)` above a subclass is **compiler error CS0160**.

### Case A — one catch, typed tests inside (what the generated docs show)

```csharp
using PaypalServerSdk.Standard.Exceptions;

try { var response = await client.{Controller}.{operation}Async(/* … */); }
catch (ApiException e)
{
    // typed payload properties — read the class for their real names
    if (e is {Typed}Exception typed) { Log(typed.ResponseCode, typed.{Field}); }
    else { Log(e.ResponseCode, e.Message); }
}
```

### Case B — a catch ladder, most specific first

```csharp
catch ({Typed}Exception e) { Log(e.ResponseCode, e.{Field}); }   // MUST come first
catch (ApiException e)     { Log(e.ResponseCode, e.Message); }   // every other non-2xx
```

## Reading the error

| Member | Type | What it holds |
| --- | --- | --- |
| `ResponseCode` | `int` | the transport status code — **on the base class**. A typed exception can declare a payload field that shadows either name: a `[JsonProperty] int? StatusCode` off the body, or — worse — `public new {Enum}? ResponseCode`, in which case `int status = e.ResponseCode;` is **`CS0266`** and reading it through the typed variable gives you an enum from the body, not the status. Whenever a typed exception is in scope, get the transport code through a base-typed variable: `((ApiException)e).ResponseCode`; `ApiResponse<T>.StatusCode` is the success-path equivalent |
| `HttpContext` | `HttpContext` | the request/response pair — **`null` when no response was received**, and `ResponseCode` is then **`-1`**, not `0` |
| `Message` | `string` | on `ApiException`: the operation's `ErrorCase` reason — the spec's description of that error, or `HTTP Response Not OK`. **Not the server's `message` field**; on a subclass see the trap below |

```csharp
if (e.HttpContext == null) { throw; }           // in the catch: nothing came back at all
var headers = e.HttpContext.Response.Headers;   // Dictionary<string, string>
```

Guard `HttpContext` before every dereference: the OAuth manager throws a bare `ApiException` with no
context when it cannot obtain a token (**csharp-authentication**). `Response.RawBody` is a `Stream`
the constructor has already read to the end to fill the typed properties — read those, or log the
exchange (**csharp-configuration-resilience**).

## Colliding exception members

A typed exception's properties come from the error schema, but it also inherits `System.Exception`'s
members. **This only arises if one of the schema's fields actually collides** — plenty of SDKs have typed
exceptions and no collision at all, in which case none of the below applies and every property is spelled
exactly as the schema names it. Check before you rely on any of it: grep `Exceptions/` for `public new`,
and for `Property` as a suffix. If neither appears, there is no collision to handle, and `e.Message` is
the SDK's reason string with no API text hiding behind it.

Where a field *does* collide, which of the two treatments you get is fixed at generation time, and this
SDK has already decided:

```csharp
// error body: { "code": "…", "message": "…" }
catch ({Typed}Exception e)
{
    var code      = e.Code;              // no collision — name unchanged; its TYPE is the schema's
    string apiMsg = e.Message;           // `public new string Message` — the API's "message"
}
```

This SDK has **no `*Property` member**: a colliding field is declared `public new string Message`,
shadowing the inherited one. Read
it through a `{Typed}Exception` variable for the API's text, and through an `ApiException` variable for
the SDK's reason string.

The names that can collide are `Message`, `ResponseCode`, `StackTrace`, `Source`, `Data`, `HelpLink`,
`HttpContext`, `InnerException`, `HResult`, `TargetSite` and the `System.Exception` methods — any of those that a schema actually declares is shadowed with `new` rather than
renamed. A name not in that list is never touched.

Never infer a member name **or its type** from the JSON: read `Exceptions/{Typed}Exception.cs` or the
**Fields** table in `doc/models/{typed-exception}.md`. A field that looks like a string in the body can be
declared `int?`, so assigning it to a `string` is `CS0029` — prefer `var`. Plain models never get *this*
rename — they have their own, an `M` prefix on a wire name that collides with a C# keyword or the
declaring type, so `"value"` emits as `MValue` (**csharp-models**).

> **An off-shape error body is only sometimes swallowed. Some shapes make the deserializer throw from
> inside the exception's own constructor — which means no `ApiException` is ever constructed and
> `catch (ApiException)` never fires.** Those escape your entire ladder. Which rows below are live depends
> on the build, so read the Outcome column rather than assuming all three throw:
>
> | Error body | Outcome |
> | --- | --- |
> | Not JSON at all (a proxy's HTML page) | **Swallowed**, as you'd hope: the typed exception is constructed with every reference property `null` and non-nullable ones at `0`, `false` or the first enum member |
> | Valid JSON, but a value outside a declared enum's domain | **`Newtonsoft.Json.JsonSerializationException`** — generated enums are strict and usually have no `_Unknown` member, so an unexpected string is fatal |
> | Valid JSON containing **any** field the schema does not declare | **Ignored.** This SDK emits no `[JsonExtensionData]` on its exception classes, so an undeclared field is silently dropped and the typed exception is constructed normally. There is no failure to handle here |
>
> A **2xx** response whose body is not the declared JSON is the same class of failure on the success path:
> a raw `Newtonsoft.Json.JsonReaderException` out of `CoreHelper.JsonDeserialize`, again outside
> `ApiException`.
>
> **One more escape route, and `JsonException` does not catch it.** Where a typed exception's payload has a
> `oneOf`/`anyOf` member, a body whose value matches no case throws `OneOfValidationException` /
> `AnyOfValidationException` from inside that exception's own construction. Those extend
> `System.Exception`, so they escape the typed arm, `catch (ApiException)` **and** a
> `catch (Newtonsoft.Json.JsonException)` arm. If a typed exception declares a union member, catch them
> explicitly (`APIMatic.Core.Types.Sdk.Exceptions`) alongside the Json arm.
>
> So: **branch on `ResponseCode`, not on payload members, and put a `catch (Newtonsoft.Json.JsonException)`
> arm in the ladder** — it covers every throwing row above plus the 2xx case, whichever of them this build
> can produce. Rows whose Outcome says *Swallowed* or *Ignored* need nothing: they arrive as an ordinary
> typed exception. One caveat before you write a test for the enum row: an SDK that declares **no enums**
> cannot produce it either. `Newtonsoft.Json` is reachable from a consumer
> with no explicit `PackageReference` — it comes in transitively. Treat hitting the arm as "the server sent
> something this SDK does not model".

## Where the exception surfaces

The `try` goes around the `await`, and the blocking twin rethrows the inner exception unchanged — so
`catch (ApiException)` works on both forms. Call `.Result`/`.Wait()` yourself and .NET wraps it in an
`AggregateException` your catch misses. For a `void` operation the
exception is the only signal there is.

## Failures that are not `ApiException`

- **Transport failures** never reach the error map and are not wrapped: connection refused, DNS
  failure and TLS errors surface from `System.Net.Http` as `HttpRequestException`, a timeout or a
  cancelled `CancellationToken` as a cancellation exception.
- **A breach of `MaximumRetryWaitTime` is `Polly.Timeout.TimeoutRejectedException`, and it is not a
  cancellation.** When it fires, the exception comes from Polly, not from
  `System.Threading`. It does **not** derive from
  `OperationCanceledException`, so a catch ladder written for cancellation misses it and the call
  escapes as an unhandled exception — a stalled provider then surfaces to your caller as a bare `500`
  rather than the timeout you meant to return. Catch it by name, or catch broadly at the boundary. The
  type comes from the runtime's dependency rather than the SDK, so it appears in no generated file:
  see **csharp-configuration-resilience** for where it is configured.
- **A credential the operation's auth group requires is missing.** For a scheme that registers a wire
  parameter, the group is validated as the request is built and the failure is `AuthValidationException`
  (an `ArgumentNullException`) listing the credentials it wanted — that one **escapes**
  `catch (ApiException)`. An OAuth grant scheme never fails that way; its failures come from the **token
  request** and all land inside `catch (ApiException)`: `ApiException` with
  `OAuth token is expired...`/`ResponseCode == -1`/`HttpContext == null` when the token has no usable
  expiry, or **`OAuthProviderException`** — a typed subclass carrying the provider's own status and a
  real `HttpContext` — when the token endpoint itself rejects the request. Order any typed catch for it
  before the base one, and fix the client construction rather than the catch block
  (**csharp-authentication**).
- **JSON deserialization of a response or an error body**, on both the success and failure paths —
  `Newtonsoft.Json.JsonSerializationException` / `JsonReaderException`, thrown before any `ApiException`
  exists. See the callout above; a `catch (Newtonsoft.Json.JsonException)` arm is the only thing that
  catches these.
- **Union (`oneOf`/`anyOf`) validation fires while a *response* is deserialized**, not while the
  request is built. A body matching no variant, or — for `oneOf` — more than one, throws
  `OneOfValidationException` (in `APIMatic.Core.Types.Sdk.Exceptions`) / `AnyOfValidationException`; both extend `System.Exception`, so they
  escape `catch (ApiException)` on the very same `await` (**csharp-models**).

## Notes

- **Retries have already happened** by the time the exception reaches you — read
  `client.HttpClientConfiguration` for what this client retries (**csharp-configuration-resilience**).
- **Do not log `HttpContext.Request` wholesale.** Per `doc/http-request.md` it exposes `Username` and
  `Password` alongside `Headers` and `Body`; log `QueryUrl`, `HttpMethod` and the status instead.
- **Do not leak SDK exception types across your own API boundary.** Translate them in one place,
  keyed off `ResponseCode` and the typed payload, keeping the original as the `InnerException`.

## Next

- Step 6, retries, timeouts, proxy, transport, logging and the base URL → **csharp-configuration-resilience**
- Step 7, stubbing the SDK → **csharp-testing**

Retry and timeout behaviour is named above only where an error surfaces it. The **defaults** — what this client does when you configure nothing — belong to **csharp-configuration-resilience**, and reading them here is not a substitute for reading them there.
