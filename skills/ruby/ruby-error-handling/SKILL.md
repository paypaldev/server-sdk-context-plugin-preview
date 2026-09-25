---
name: 'ruby-error-handling'
description: 'Handle errors from the PayPal Server SDK Ruby SDK. Load before your first `rescue` around a call, or when building an error-translation layer. The class list won''t tell you which statuses this SDK actually raises on, where the error body''s fields live, or that the `doc/` Errors table names classes that may not exist here.'
---

# Error handling for an APIMatic Ruby SDK

Endpoint methods **raise** on error responses. Everything they raise descends from
`PaypalServerSdk::APIException`, which extends `CoreLibrary::ApiException` from the `apimatic_core` gem.

> ### ⚠ `rescue PaypalServerSdk::APIException` is not a complete net
>
> **A typed exception can fail while being constructed, and what escapes is not an `APIException`.**
> A generated `{Name}Exception#initialize` deserialises the body unguarded:
>
> ```ruby
> def initialize(reason, response)
>   super(reason, response)
>   hash = APIHelper.json_deserialize(@response.raw_body)   # raises here
>   unbox(hash)
> end
> ```
>
> `APIHelper.json_deserialize` raises `TypeError, 'Server responded with invalid JSON.'` for a body it
> cannot parse. That happens *inside the constructor*, before the exception object exists, so nothing
> derived from `APIException` is ever raised and your `rescue` does not run.
>
> The trigger is not exotic: it is an HTML error page from a gateway or WAF in front of the API, an
> empty body, or a bare scalar — exactly the `502` and `504` shapes a stalled provider produces, and so
> exactly the case a resilience path is written for. Add `rescue TypeError` alongside, or rescue
> `StandardError` at the boundary, and treat a non-idempotent call that ends this way as **outcome
> unknown** — the request was sent and the response was unreadable.



## Which exception does an operation raise?

Two shapes, and the difference is generated per operation:

- **Typed subclass** — where a status code has an error model, there is a class under
  `lib/paypal_server_sdk/exceptions/` (`class {OperationException} < APIException`) and the
  operation registers it for that status. The instance carries the error body's fields as ordinary
  attributes.
- **Base `APIException`** — everything else.

Which shape an operation uses is a property of this API's definition, not a rule of thumb: an SDK can
register a typed class for every documented status *and* for the default case, leaving no operation that
raises the base class. Read the registrations rather than assuming either shape — a bare
`rescue APIException` still rescues a typed instance, and then reads none of its attributes.

You do not have to guess. Two places tell you:

1. **`doc/controllers/{controller}.md`** — each operation has an **Errors table** mapping HTTP status
   code → description → exception class. Read it first, then confirm each class against
   `ls lib/paypal_server_sdk/exceptions/`.
2. The operation in the controller folder under `lib/paypal_server_sdk/` — where this build registers
   errors at all (the callout below settles that), its response handler holds a
   `.local_error('{status}', '{message}', {ExceptionClass})` line per documented error, and a
   `GLOBAL_ERRORS` constant on the controller base class supplies the catch-all. The folder
   (`controllers/`, `apis/`, …) and the base class (`BaseController`, `BaseApi`, …) are both named per
   SDK, the latter after its controller postfix, so
   `grep -rn GLOBAL_ERRORS lib/paypal_server_sdk/` for the real constant rather than assuming either name.

> **Both halves of that registration exist because this SDK raises for HTTP error statuses.** The client
> wires the `GLOBAL_ERRORS` constant into its global configuration — the
> `'default'` entry is what turns *any* unsuccessful response into `APIException` — and each response
> handler carries its own `.local_error(...)` lines for the statuses the API documents. **The two stand or
> fall together**, so they are present or absent as a pair: there is no build in which only the declared
> statuses raise. `grep -rn global_errors lib/paypal_server_sdk/client.rb` shows the wiring.

## Rescue the exception
Ruby matches `rescue` clauses top-down, and every typed exception is a **subclass** of `APIException`.
Put the typed clauses first — an `APIException` clause above them swallows everything:

```ruby
require 'paypal_server_sdk'

begin
  result = client.{controller_name}.{operation}
  # use result
rescue PaypalServerSdk::{OperationException} => e
  # typed: the error body's fields are attributes on e — read the class under exceptions/ for the names
  warn "typed error: #{e.{error_attribute}}"
rescue PaypalServerSdk::APIException => e
  # everything else the API returned
  warn "API error: #{e}"
end
```

What you can read off the raised object:

- **`e.message`** — on the base `APIException`, the reason the exception was raised with (the
  `error_message` registered for that status, e.g. `'HTTP response not OK.'` for the global catch-all),
  *not* the API's error text. **On a typed subclass this inverts:** when the error model happens to have a
  `message` field, the generated `attr_accessor :message` shadows `Exception#message`, so `e.message`
  returns the API's error text — and, when the body carried no `message`, the `nil`-or-`SKIP` value
  described below rather than the base class's reason. Check the class under `exceptions/` for an
  `attr_accessor :message` before relying on either meaning.
- **`e.to_s` / `e.inspect`** — on the base `APIException`, a summary including the **status code** and
  reason, so `puts e` is the fastest way to see what came back. Typed subclasses **override both** to
  print their unboxed model fields only — no status code, empty when the body did not parse, and
  `#<Object:0x...>` for each field the `SKIP` sentinel below is holding. Use the routes below when you
  need the status from a typed error.
- The base class also carries the reason and the underlying `HttpResponse`. **Read
  `lib/paypal_server_sdk/exceptions/api_exception.rb` and the `CoreLibrary::ApiException` it extends for
  the exact accessor names** before reaching for the status code or the raw body off the exception —
  they come from the gem, not from generated code.
- **Typed subclasses only** — one attribute per field of the error model, populated by `unbox` from the
  parsed response body in the constructor. `doc/models/{operation-exception}.md` lists them.
- A typed exception is only populated from a response body that parses as the error model, and **an
  unset attribute is not always `nil`**. When the body is not JSON at all, `unbox` returns before
  assigning anything and every attribute reads `nil`. When it parses but a key is missing, only
  attributes the error model marks *required* fall back to `nil`; an **optional** one is left holding the
  class's private `SKIP` sentinel — a bare `Object`, so `e.{error_attribute}.nil?` is never true,
  `e.{error_attribute}&.each` raises `NoMethodError`, and interpolating it prints `#<Object:0x...>`.
  `SKIP` is a `private_constant`, so you cannot compare against it: read the class's `unbox` under
  `exceptions/` to see which attributes get `nil` and which get the sentinel, and guard by type
  (`e.{error_attribute}.is_a?(String)`) rather than by nil check.

### Getting the status code reliably

If you need the status code as data rather than as text, take it from a place that documents it:
- **`ApiResponse#status_code`** — this SDK returns complete responses, so every successful non-paginated
  call hands you the status code directly (see **ruby-calling-endpoints**). `ApiResponse` also answers
  `success?` / `error?` and exposes `errors`, so you can branch on the response instead of rescuing.
- **`http_callback`** — an `HttpCallBack` whose `on_after_response(response)` receives the
  `HttpResponse`, which documents `status_code`, `reason_phrase`, `headers` and `raw_body`. This fires
  for successes and failures alike, which makes it the seam the generated tests use (see
  **ruby-testing**).

## Do not swallow non-API failures

`rescue PaypalServerSdk::APIException` catches API errors only. Transport failures — connection refused,
DNS failure, TLS error, a timeout — come out of **Faraday**, not the SDK, and are not `APIException`.
Rescuing `StandardError` around a call to be safe will therefore hide bugs in your own code as well;
rescue the Faraday error classes explicitly if you need to handle transport failures, and let anything
else propagate.

## Notes

- Retries for transient statuses happen **before** the exception is raised, and only for the status
  codes and HTTP methods baked into this SDK's `Configuration` defaults. Those defaults commonly exclude
  `POST`/`PATCH`/`DELETE`, whose errors then surface with no retry at all — see
  **ruby-configuration-resilience**.
- A call can also fail **before any request is sent**, when the configured auth scheme cannot be
  satisfied. What reaches you is `CoreLibrary::AuthValidationException`, from the `apimatic_core` gem —
  **not** an `APIException` and **not** a Faraday error, so the ladder above does not rescue it and it
  escapes as an unhandled exception. Give it its own `rescue`, or let it escape as the configuration bug
  it is. Three consequences:

  1. Raised **before the request is sent**, so a non-idempotent call definitely did not happen — unlike
     a Faraday timeout, where the outcome is unknown.
  2. Its message is the scheme's fixed `error_message`, so a wrong credential, an expired one and a
     token endpoint that is down all read identically. Grep that string in
     `lib/paypal_server_sdk/http/auth/` to find the scheme it came from.
  3. For an **OAuth grant** the real cause is discarded on the way: the automatic pre-call token fetch
     rescues the provider's own exception, hands back the previously held token — `nil` on a first call
     — and the fixed message above is all that survives. Call `fetch_token` **explicitly at startup** to
     get the SDK's OAuth provider exception instead, carrying the token endpoint's status and body.

  See **ruby-authentication**.
- When you translate SDK errors into your own domain errors, do the translation once in a wrapper around
  the client rather than at every call site, and keep the original exception as the cause.

## Next

- Step 6, retries, timeouts, transport, logging and the base URL → **ruby-configuration-resilience**
- Step 7, stubbing the SDK → **ruby-testing**

Retry and timeout behaviour is named above only where an error surfaces it. The **defaults** — what this client does when you configure nothing — belong to **ruby-configuration-resilience**, and reading them here is not a substitute for reading them there.
