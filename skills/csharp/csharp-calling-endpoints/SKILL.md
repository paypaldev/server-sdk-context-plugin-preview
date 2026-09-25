---
name: 'csharp-calling-endpoints'
description: 'Call operations on the PayPal Server SDK C# SDK. Load before the first call, and when building a request body or reading a response. The signature won''t tell you controllers are properties on the client, that sync and async forms differ in which one takes a `CancellationToken`, or what an optional parameter''s default actually puts on the wire.'
---

# Calling endpoints on an APIMatic C# SDK

> **`doc/` is in the SDK's repository, not in the installed package.** Every pointer to
> `doc/controllers/` or `doc/client.md` below assumes you have the repository checked out. Where you
> only have the installed artifact — the common case — the equivalent source is the operation's own signature in `Controllers/`, and it is
> authoritative in a way the docs are not.

Operations are methods on a **controller you read off the client as a property** — you never construct
one:

```csharp
using PaypalServerSdk.Standard;
using PaypalServerSdk.Standard.Controllers;        // only when you spell the controller type instead of `var`

var client = new PaypalServerSdkClient.Builder()./* ... */.Build();
var controller = client.{Controller};  // the property's declared type varies per build
var response = await controller.{Operation}Async(/* ... */);
```

The property is named after the controller class and is backed by a `Lazy<T>`, so the instance is
created on first access and shared thereafter (see **csharp-client-initialization**). Every
controller's constructor is **`internal`**, so `new {Controller}(...)` does not compile. **The class
suffix varies per SDK** — some builds carry none — so read the names off the **Controllers**
table in `doc/client.md`.

## Two method shapes — blocking and `{Operation}Async`

Each operation is generated as a pair, side by side in the same file:

```csharp
public {ReturnType} {Operation}({params})
public async Task<{ReturnType}> {Operation}Async({params}, CancellationToken cancellationToken = default)
```

- On a non-paginated operation the sync form is a one-line `CoreHelper.RunTask(...)` delegation that
  **blocks the calling thread** and drops the `CancellationToken`, so **a sync call cannot be
  cancelled**. A server-sent-events operation gets the `...Async` form only.
- Apart from the `CancellationToken` there is **no per-call request-options argument**: no per-call
  timeout, no arbitrary-header bag. Those are client-level — see **csharp-configuration-resilience**.
- Non-2xx responses **throw** `ApiException` or a typed subclass — see **csharp-error-handling**.
  `doc/controllers/{group}.md` prints **only the `...Async` signature**; open the controller `.cs` file
  for the blocking twin.

## Parameters — positional by default, with the spec's own defaults

```csharp
public async Task<ApiResponse<{Model}>> {Operation}Async(
        string {pathParam},                 // required parameters first, with no default
        Models.{RequestModel} body,         // request body — tagged `Body` in the doc's table
        string {xSomeHeader} = null,        // optional header
        int? limit = 25,                    // optional query param — a real default, not null
        CancellationToken cancellationToken = default)
```

- Use **named arguments** to skip over optionals: `controller.{Operation}Async(id, limit: 100)`.
  Optional enum and value-typed parameters are nullable (`{EnumType}?`, `DateTime?`, `int?`).
- **An operation may instead collapse its parameters into one input model**, becoming
  `{Operation}Async({Operation}Input input, CancellationToken cancellationToken = default)` — note the
  controller declares that type **unqualified**, even though you need the `Models` alias to name it from
  your own code. It happens when the operation has **more than one** non-constant parameter *and* either
  this build collapses them, or that operation was built to collapse its own, so
  the two forms mix freely inside one controller and you must **read each signature**. The name is usually
  `{Operation}Input`, but a spec-supplied collection name drops the `Input` suffix and a clash across
  groups prefixes the group — so take it from the signature, never build it.
- **An optional parameter's default is not always `null`.** The declared default is also re-applied on
  the wire, so passing `null` sends the default rather than dropping the parameter.
- **`DateTime` parameters are formatted per parameter** — one may go out as a full ISO-8601 timestamp
  and another in the same SDK as `yyyy-MM-dd`; read the `.Query(...)` line when it matters. An endpoint
  accepting unmodelled query or form values also carries a trailing `Dictionary<string, object>` bag
  before the token.

## Building request models

```csharp
using Models = PaypalServerSdk.Standard.Models;   // alias, not a plain import — see csharp-models

var body = new Models.{Model} { {RequiredProp} = value, {OptionalProp} = value };
```

**The bare `Models.{Model}` you see in generated source and in `doc/` does not compile from your code** —
`Models` is nested inside `PaypalServerSdk.Standard`, and a `using PaypalServerSdk.Standard;` does not bring nested
namespaces into scope, so it is `CS0246`. The alias above is what makes the same spelling work, and it is
also what you want when a model name shadows a BCL type. Importing `PaypalServerSdk.Standard.Models` directly
works too, but then you write `{Model}` unqualified and inherit any BCL collision.

**Data models have no `Builder`** in the default shape — unlike the auth credential models and
`HttpClientConfiguration`. Use the **object initializer**, as `doc/controllers/{group}.md` does. For
optionality, unions, collections, dates and the immutable variant that *does* carry a `Builder`, load
**csharp-models**.

## Enums

Enum-typed parameters and fields take plain C# `enum` members (`{EnumType}.{Member}`), and the SDK puts
the `[EnumMember]` wire value on the wire for you. **Read `Models/{EnumType}.cs` for the real member
names** — never derive them from the wire strings. The declaration shapes are in **csharp-models**.

## File uploads — `FileStreamInfo`

```csharp
using PaypalServerSdk.Standard.Http.Client;

using (var stream = File.OpenRead("./{file}"))
{
    await controller.{Operation}Async(
        {pathParam}, new FileStreamInfo(stream, "{file}", "application/pdf"));
}
```

**`FileStreamInfo` lives in `PaypalServerSdk.Standard.Http.Client`, not in `.Models`** — the import people get
wrong. Only the stream is required. The other multipart fields are **ordinary parameters on the same
method**, tagged `Form` in the doc's table — there is no parts collection to assemble.

## Reading the response


> Every non-void, non-paginated operation in this SDK
> returns an `ApiResponse<T>` envelope. That applies to the whole SDK at once.

```csharp
using PaypalServerSdk.Standard.Http.Response;

ApiResponse<{Model}> response = await controller.{Operation}Async({pathParam});
int status = response.StatusCode;
Dictionary<string, string> headers = response.Headers;
{Model} value = response.Data;              // the deserialized payload
```

Those are its only three properties: the payload is on **`.Data`** — not `.Result`, not `.Body`.

An operation with **no response body** returns `void` / a bare `Task` — never a wrapped one — so a
`DELETE` gives you **no status code at all**. Register an `HttpCallback` on the client when you need
one: see **csharp-configuration-resilience**.

### A `dynamic` (or `object`) return

An operation with no modelled response type is declared `Task<dynamic>` **or `Task<object>`** — which of
the two varies per build, so grep for both (`grep -c 'Task<dynamic>\|Task<object>' Controllers/*.cs`)
and expect it to be most of the surface on some SDKs. Everything below applies to either — and `dynamic` here does **not**
mean a deserialized object. The SDK hands back the **raw `System.IO.MemoryStream`** of the body, buffered
and positioned at 0. Member access on it throws `RuntimeBinderException`, and
`ApiHelper.JsonSerialize(result)` throws too. Read it instead:

```csharp
object raw = await controller.{Operation}Async(/* … */);   // cast to object at the boundary

if (raw is not System.IO.Stream body) { /* empty body — see below */ }
else
{
    using var reader = new StreamReader(body);
    var token = Newtonsoft.Json.Linq.JToken.Parse(await reader.ReadToEndAsync());
    var obj = token as Newtonsoft.Json.Linq.JObject;    // null when the root is an array
}
```

Four things that are not obvious and each cost a round:

- **An empty body yields `null`, not an empty stream** — a 204 or a no-content 200 returns `null`. The
  reference cast itself succeeds on `null`; what throws is the first thing you hand it to, e.g.
  `new StreamReader(null)` → `ArgumentNullException`. Null-check before reading.
- **Use `JToken.Parse`, not `JObject.Parse`** — a top-level JSON array is legal and `JObject.Parse` throws
  `JsonReaderException` on it. Narrow with `as JObject` before indexing by name; indexing a `JArray` by
  string throws `ArgumentException` rather than returning null.
- **Assign to `object`, not `dynamic`** — leaving it `dynamic` makes every downstream call dynamically
  dispatched, and any lambda in that chain is `CS1977`.
- **`using var reader` disposes the stream**, so the body can be read once. Pass `leaveOpen: true` if you
  also want to log the raw text.

### A binary body

A binary/file response deserializes to `System.IO.Stream` — there is no `byte[]` overload — so copy it
out yourself:
`await response.Data.CopyToAsync(destination)`.

## Pagination — none in this SDK

**This API declares no paginated operation.** The SDK ships no pagination utilities, no
`Pageable<,>`/`AsyncPageable<,>` types, and no operation returns one. A list endpoint here returns the
ordinary type described above: drive its own page/cursor parameters yourself and stop on whatever the
API uses to signal the end.

## Worked example — a list/GET call

```csharp
// Signature (illustrative — read the real one):
//   {Operation}Async(string {pathParam}, DateTime? {since} = null,
//                    CancellationToken cancellationToken = default)
try
{
    ApiResponse<List<{Item}>> response = await client.{Controller}.{Operation}Async(
        {pathParam}, {since}: DateTime.UtcNow.AddDays(-7), cancellationToken: token);

    foreach (var item in response.Data) { Use(item.{Field}); }
}
catch (ApiException e) { /* see csharp-error-handling */ }
```

## Finding the right method in the SDK source

- `doc/controllers/{group}.md` — **grep here first.** It names the controller class and the
  `client.{Controller}` line that gets it, then per operation a parameters table (type,
  `Template`/`Query`/`Header`/`Form`/`Body`, required vs optional, the default), the response type, a
  sample, and — **when the SDK emits one** — an **Errors** table mapping each status to an exception class.
  Some SDKs emit no `## Errors` section at all and register their errors on the controller base class
  instead, so treat the table as a finding aid and the `.cs` as the authority (**csharp-error-handling**).
- The controller `.cs` file — authoritative signatures for **both** forms, the real defaults, and the
  request builder showing how each parameter is sent and which auth schemes apply.
- `doc/client.md` — the **Controllers** table.

## Next

- Unions, collections, dates, nullable-optional fields → **csharp-models**
- Errors and status codes → **csharp-error-handling**
- Retries, timeouts, proxy, logging → **csharp-configuration-resilience**
