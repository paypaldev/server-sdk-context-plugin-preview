---
name: 'csharp-testing'
description: 'Unit-test code that calls the PayPal Server SDK C# SDK. Load before stubbing the SDK. The member list won''t tell you the one injectable seam is an `HttpMessageHandler` passed through `.HttpClientConfig(...)`, that the client is `sealed` with an internal controller constructor, or the two things a stub handler must do before a call gets through at all.'
---

# Testing code that uses an APIMatic C# SDK

The SDK ships **no mocking helpers and no in-memory transport**. The one injectable seam is the
`HttpClient` it calls through — `.HttpClientConfig(config => config.HttpClientInstance(new
HttpClient(handler)))`, with an `HttpMessageHandler` you write. Confirm both members first:
`HttpClientConfig(Action<HttpClientConfiguration.Builder>)` in `PaypalServerSdkClient.cs` and
`HttpClientInstance(HttpClient, bool overrideHttpClientConfiguration = true)` in
`Http/Client/HttpClientConfiguration.cs`.

The samples below use xUnit for reference only — mirror whatever the project already uses.

## What you cannot fake — and what to do instead

**This SDK emits no interfaces, so there is nothing for a mocking library to
substitute.** The client is `sealed` with a `private`
constructor, its only interface exposes configuration but no operations, and every operation is a
plain **non-`virtual`** method behind an `internal` constructor.

So put a narrow interface of your own in front of the SDK, implement it over the client — calling
`_client.{Controller}.{operation}Async(id, cancellationToken: ct)` and returning whatever your callers
need — and test *that* against the stub handler below. See **csharp-client-initialization**.

Faking the SDK's own types is cheap — they all have public constructors:
`ApiResponse<T>` (`public ApiResponse(int, Dictionary<string, string>, T)`), every model, and every typed exception (`public {Error}Exception(string reason, HttpContext context)`).

## A reusable stub handler

```csharp
internal sealed class StubHandler : HttpMessageHandler
{
    private readonly int _status;
    private readonly string _body;
    public StubHandler(int status, string body) { _status = status; _body = body; }
    public HttpRequestMessage LastRequest { get; private set; }
    public string LastRequestBody { get; private set; }

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken cancellationToken)
    {
        LastRequest = request;
        LastRequestBody = request.Content == null ? null : await request.Content.ReadAsStringAsync();
        return new HttpResponseMessage((HttpStatusCode)_status)
        {
            Content = new StringContent(_body, Encoding.UTF8, "application/json"),
            RequestMessage = request,                      // not optional — see below
        };
    }
}

static PaypalServerSdkClient ClientOver(StubHandler handler) =>
    new PaypalServerSdkClient.Builder()
        // setter, model and argument list: copy them from the SDK map in csharp-getting-started
        .{Scheme}Credentials(new {Scheme}Model.Builder(/* the map lists the required args */).Build())
        .HttpClientConfig(config => config
            .HttpClientInstance(new HttpClient(handler))   // the only transport seam
            .NumberOfRetries(0))                           // a stubbed 5xx fails on the first attempt
        .Build();
```

Nothing leaves the process — the handler forwards nowhere. Two lines above look optional and are not:

- **Copy `request` onto the response's `RequestMessage`.** The retry policy inspects the response's
  originating request, so a stub that leaves it null fails *every* call with a
  `NullReferenceException` from inside the runtime — `NumberOfRetries(0)` does not spare you.
- **Set credentials for every scheme the operation declares** — and know which failure to expect, because
  the two kinds differ and asserting the wrong one is a false test:
  - *Parameter-registering schemes* (custom header/query, basic, static bearer) are validated while the
    request is built, so a credential-less client throws `AuthValidationException` **before your handler
    is reached** — assert zero calls on the stub.
  - *OAuth 2 grant schemes* always pass that validation, so **there is no credential-less failure to
    assert** — see the next bullet. Whether the call throws depends on what your stub answers the token
    request with, not on whether you set credentials.

  Check first: if no controller contains `WithAuth`/`WithOrAuth`/`WithAndAuth`, no operation requires a
  scheme and a credential-less client works — the configured value never reaches the wire.
- **An OAuth scheme's token request must be answered by your stub**, or the first real call fails on the
  token step. **Read the real path off the OAuth controller** — `grep -n 'Setup(HttpMethod' ` the
  authorization controller in Controllers; it varies (`/oauth/token`, `/v1/oauth/token`,
  `/v1/oauth2/token` are all real) and a wrong guess makes every stubbed test fail on the token step.
  Route on that path and return
  `{"access_token":"stub-token","token_type":"Bearer","expires_in":3600}`.

  **That stub makes a credential-less client succeed**, because the grant never checks that you supplied a
  client id — so a test written as "no credentials ⇒ throws" *fails*. Pick the outcome deliberately:
  answer the token request to test the happy path; omit `expires_in` to get
  `ApiException("OAuth token is expired...")`; answer it with an error status to get
  `OAuthProviderException`. Assert on what you stubbed, and set credentials whenever the test is about
  anything other than the token exchange.

## Test a success path

```csharp
var client = ClientOver(new StubHandler(200, "{\"{wireName}\": \"expected\"}"));

ApiResponse<{Model}> response = await client.{Controller}.{operation}Async(/* args */);
Assert.Equal(200, response.StatusCode);
Assert.Equal("expected", response.Data.{Property});
```

Operations in this SDK hand back an `ApiResponse<T>`
wrapper and the status code is assertable directly.

Assert on the **deserialized model** — that is what exercises the SDK's mapping. An operation with no
payload breaks the pattern: it returns a bare `Task`, so there is no status code to assert (use
`HttpCallback`, below).

## Test an error path

Which exception an operation throws is per-operation: the **Errors** table in
`doc/controllers/{group}.md` names the class per status, and the `CreateErrorCase(...)` calls in that
operation's own `ResponseHandler` are the authority — see **csharp-error-handling**.

```csharp
var client = ClientOver(new StubHandler(404, "{\"code\":\"not_found\",\"message\":\"bad input\"}"));

var e = await Assert.ThrowsAsync<{Error}Exception>(
    () => client.{Controller}.{operation}Async(/* args */));

// NOT StatusCode — and go through the base type: a typed exception can shadow
// ResponseCode itself with a payload enum, making `e.ResponseCode` not an int.
Assert.Equal(404, ((ApiException)e).ResponseCode);   // see csharp-error-handling
```

**Assert the concrete type, not `ApiException`.** Every typed exception derives from it, so a test
expecting `ApiException` passes for all of them and proves nothing.

**Read the class before asserting on a payload field.** Where a typed exception declares a `message`
field, the generator emits it as `public new string Message` — so `e.Message` on the *typed* variable
returns the **API's** message from the body, and the SDK's own reason string is on the base, reachable
only as `((ApiException)e).Message`. Assert whichever you actually mean, and get there through a
base-typed variable when you want the reason. **csharp-error-handling** covers the same shadowing for
`ResponseCode`, which fails harder. Test
the `…Async` overload; the sync one only wraps it.

## Assert the outgoing request

```csharp
var handler = new StubHandler(200, "{}");
await ClientOver(handler).{Controller}.{operation}Async(/* args */);

Assert.Equal(HttpMethod.Post, handler.LastRequest.Method);
Assert.EndsWith("/expected/path", handler.LastRequest.RequestUri.AbsolutePath);
Assert.Contains("expectedParam=value", handler.LastRequest.RequestUri.Query);
Assert.Contains("\"expectedField\":\"value\"", handler.LastRequestBody);
```

Worth doing once per model you build: an enum travels as its `[EnumMember]` value, and an
optional-and-nullable field serializes from the moment you assign it, `null` included (see
**csharp-models**). `Content-Type` sits on `request.Content.Headers`.

## The other seam — `HttpCallback` observes, it cannot stub

```csharp
var callback = new HttpCallback();
var client = new PaypalServerSdkClient.Builder().HttpCallback(callback).Build();

await client.{Controller}.{operation}Async(/* args */);   // this really goes out

Assert.Equal(204, callback.Response.StatusCode);          // also .Headers, .RawBody and .Body
Assert.Equal(HttpMethod.Delete, callback.Request.HttpMethod);
```

`HttpCallback` is a **concrete class, not an interface**: a bare instance records the last exchange, and
subclassing it overrides `OnBeforeRequest`/`OnAfterResponse` to see every one. `Request.HttpMethod`,
`Request.QueryUrl`, `Response.StatusCode`, `Response.Headers` and `Response.RawBody` are the members to
assert on; most of them come from `APIMatic.Core`, so the SDK's own `HttpRequest.cs` looks empty.

It cannot substitute a response, but it **composes with the handler seam**: register the callback *and*
an `HttpClientInstance` over your stub handler and it observes the stubbed exchange without anything
leaving the process. On an SDK that returns bare `T` that is the only way to assert a status code in a
unit test.

## Notes

- **Set `NumberOfRetries(0)` in every stubbed test**, so a stubbed `5xx` fails on the first attempt
  instead of sitting through backoff waits. Never assume what the policy would otherwise have been —
  see **csharp-configuration-resilience**.
- **Do not assume `Assert.Equal(expectedModel, actual)` compares by value — and finding an `Equals`
  override is not proof that it does.** Models commonly declare `public override bool Equals` while
  **ANDing in `base.Equals(obj)`**; where the SDK emits `Models/BaseModel.cs`, that base overrides neither
  `Equals` nor `GetHashCode`, so the chain bottoms out in **reference equality** and two identically
  constructed models are never equal. Check the base as well as the model. The two move
  independently: some SDKs emit `Equals` on models and some emit none, while `GetHashCode` appears only
  on immutable-models builds and is usually absent even where `Equals` is present — which silently
  breaks any `HashSet` or dictionary keyed on a model. Exception classes get neither. **Prefer asserting field by field, or comparing
  `ApiHelper.JsonSerialize(...)` of both** (`PaypalServerSdk.Standard.Utilities`), rather than relying on
  equality at all.
- **Build each test's client from `new PaypalServerSdkClient.Builder()`.** `ToBuilder()` drops the HTTP
  client configuration, so a client cloned from a stubbed one talks to the real network
  (**csharp-client-initialization**).
- **`client.GetBaseUri()` asserts base-URL resolution without sending anything** — it catches a wrong
  `Environment` or a wrong configuration variable in the URL template.
- **The SDK's own test project is not a template for yours** — its generated fixture builds a real
  client from environment variables and calls the live API. Many SDKs ship without one; treat its
  absence as normal.

## Next

That is the last step of the workflow. If you have not read **csharp-configuration-resilience** yet, read it before you ship — its subject is the client's defaults, which apply whether or not you set anything.
