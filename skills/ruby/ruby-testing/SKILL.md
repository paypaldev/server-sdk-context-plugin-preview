---
name: 'ruby-testing'
description: 'Unit-test code that calls the PayPal Server SDK Ruby SDK. Load before stubbing the SDK. The argument list won''t tell you the seam is a `Faraday::Connection` you build yourself and pass as `connection:`, that `http_callback` observes rather than substitutes, or that a stubbed 5xx needs `max_retries: 0` to fail on the first attempt.'
---

# Testing code that uses an APIMatic Ruby SDK

The SDK has **no mocking helpers for consumers**. Its transport is **Faraday**, and the `Client`
constructor exposes two relevant keyword arguments:

- **`connection:`** — a `Faraday::Connection` you build yourself, used for every request. Building it
  with **Faraday's test adapter** is the in-process seam: no network, no global patching.
- **`http_callback:`** — an `HttpCallBack` whose `on_before_request` / `on_after_response` hooks
  **observe** the request and response. This is what the SDK's own generated tests use for assertions —
  it cannot stub anything.

The samples below use Minitest for reference only — mirror whatever the project already uses.

## A reusable stub helper

```ruby
require 'faraday'
require 'paypal_server_sdk'

def client_returning(status, body, path)
  stubs = Faraday::Adapter::Test::Stubs.new
  stubs.get(path) do
    [status, { 'Content-Type' => 'application/json' }, JSON.generate(body)]
  end

  connection = Faraday.new do |builder|
    builder.adapter(:test, stubs)
  end

  client = PaypalServerSdk::Client.new(
    connection: connection,
    max_retries: 0        # a stubbed 5xx fails on the first attempt, with no backoff wait
  )

  [client, stubs]
end
```

Faraday's test adapter is Faraday's own API, not the SDK's — check the Faraday version the SDK's gems
resolve to for the exact stub syntax, and use `stubs.verify_stubbed_calls` to assert every stub was hit.

**A scheme that fetches a token sends that request through this same connection, so stub its path too.**
The generated auth handler builds its own controller from the configuration it was given
(`@_o_auth_api = OAuthAuthorizationController.new(config)` in `PaypalServerSdk/http/auth/`), which means
the token call rides the `connection:` you just replaced — it does not bypass the seam. A helper that
stubs only the operation's path therefore fails before the operation is reached, and **it does not fail
with anything that names your test**: Faraday raises
`Faraday::Adapter::Test::Stubs::NotFound`, and the handler's `rescue ApiException` does not catch it,
because a Faraday stub miss is not an `ApiException`. So it propagates raw instead of turning into the
handler's own `'... OAuthToken is undefined or expired.'` message. Supplying dummy credentials does not
help; the request is still made.

Stub the token path as well — with the **verb the token endpoint actually uses**, which is `post`, not
the `get` the helper above registers. Read the real path off the generated OAuth authorization
controller rather than assuming it, since it comes from the spec:

```ruby
stubs.post('{tokenPath}') do               # read the real path off the OAuth controller
  [200, { 'Content-Type' => 'application/json' },
   JSON.generate('access_token' => 'stub-token', 'token_type' => 'Bearer', 'expires_in' => 3600)]
end
```

With both registered, the calls arrive in order — the token exchange first, then the operation — so
anything you assert about "the request" has to select the one you mean rather than assume there was only
one. The alternative is to seed a token on the credentials object so no fetch happens at all; see
**ruby-authentication** for the field, and check how that SDK decides expiry before relying on it.

## Test a success path

```ruby
def test_returns_deserialized_body
  client, = client_returning(200, [{ 'id' => 123, 'name' => 'Rex' }], '/pets')

  result = client.{controller_name}.{operation}

  assert_equal 123, result.data.first.{attribute}
end
```
**Reach through `.data`.** This SDK returns complete responses, so an operation hands back an `ApiResponse`
and the payload is one level down — `result.first` on the wrapper raises `NoMethodError`, and asserting
`result.status_code` alone passes without ever exercising the deserializer. Assert on `result.data`, and
assert the status code off the same object rather than through a callback.

Assert on the **deserialized model**, not the JSON — that is what exercises the SDK's mapping.



## Test an error path



Endpoint methods raise on error responses (see **ruby-error-handling**). Assert the **specific** class
the operation declares — the Errors table in `doc/controllers/{controller}.md` names it — rather than the
`APIException` base, or the test will pass for the wrong reason:

```ruby
def test_raises_typed_exception
  client, = client_returning(422, { 'code' => 422, 'message' => 'bad input' }, '/pets')

  error = assert_raises(PaypalServerSdk::{OperationException}) do
    client.{controller_name}.{operation}
  end

  assert_equal 'bad input', error.{message_attribute}
end
```

For an operation with no declared error model, assert `PaypalServerSdk::APIException` instead.

## Assert the outgoing request

Faraday's test stub block receives the request environment, so capture it there:

```ruby
def test_sends_correct_request
  captured = nil
  stubs = Faraday::Adapter::Test::Stubs.new
  stubs.post('/pets') do |env|
    captured = env
    [201, { 'Content-Type' => 'application/json' }, '{}']
  end
  connection = Faraday.new { |builder| builder.adapter(:test, stubs) }
  client = PaypalServerSdk::Client.new(connection: connection, max_retries: 0)

  client.{controller_name}.{operation}(body)
  # collapsed build (def {operation}(options = {})): {operation}('body' => body) — see ruby-calling-endpoints

  assert_equal :post, captured.method
  assert_includes captured.url.path, '/pets'
  assert_equal 'value', JSON.parse(captured.body)['expectedField']
end
```

Asserting the serialized body is worth doing at least once per model you build: an attribute you omitted,
or one whose wire name differs from the Ruby name, disappears silently (see **ruby-models**).

## Observe with `http_callback` instead

When you only need the status code, headers or raw body of a real (or stubbed) call, use the same
`HttpCallBack` pattern the SDK's generated tests use:

```ruby
class ResponseCatcher < PaypalServerSdk::HttpCallBack
  attr_reader :response

  def on_before_request(request); end

  def on_after_response(response)
    @response = response
  end
end

catcher = ResponseCatcher.new
client = PaypalServerSdk::Client.new(http_callback: catcher)

client.{controller_name}.{operation}

assert_equal 200, catcher.response.status_code
assert_equal expected_json, JSON.parse(catcher.response.raw_body)
```

`HttpResponse` exposes `status_code`, `reason_phrase`, `headers`, `raw_body` and `request`. On this SDK it is
a convenience rather than a necessity — complete responses are enabled, so a success path already gives you
`result.status_code` directly — but it is the only route to the raw body of a call whose payload was
deserialized, and it fires on failures too, so it is also how you inspect what an exception was raised
from.

## Notes

- **Disable retries in tests** (`max_retries: 0`) so a stubbed 5xx fails on the first attempt instead of
  waiting out the backoff. To test that retries *do* fire, stub the same path twice (503 then 200) and
  assert both stubs were consumed — but first check `retry_methods` in
  `lib/paypal_server_sdk/configuration.rb`, since a `POST` is commonly not retried at all.
- **A stubbed client is all you need.** Controllers are memoized readers on the client, not objects you
  construct, so there is nothing else to fake.
- **Substitute the client, not the controller.** Inject the `Client` (or your own wrapper over it) into
  the code under test and pass a stubbed one in tests — see the dependency-injection section of
  **ruby-client-initialization**.
- If the project already uses **WebMock** or **VCR**, stub at that level instead; the assertions above
  apply unchanged, and you keep one HTTP-stubbing mechanism across the suite.
- The SDK's own `test/` directory (present only in an SDK that ships tests) is a working
  reference: `test/http_response_catcher.rb` plus a Minitest base class that builds the client with
  `Client.from_env(..., http_callback: HttpResponseCatcher.new)`.

## Next

That is the last step of the workflow. If you have not read **ruby-configuration-resilience** yet, read it before you ship — its subject is the client's defaults, which apply whether or not you set anything.
