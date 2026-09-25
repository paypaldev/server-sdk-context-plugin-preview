---
name: 'ruby-configuration-resilience'
description: 'Tune the PayPal Server SDK Ruby SDK client — retries, timeouts, transport, logging and the base URL. Load before changing any transport setting. The argument list won''t tell you the retry default is fixed when the SDK is generated, that `timeout` covers each attempt rather than the call, or what a stdlib `Logger` does to the SDK''s log parameters.'
---

# Configuration & resilience for an APIMatic Ruby SDK

Every setting is a **keyword argument passed at construction** (see **ruby-client-initialization**).
There is no options object and nothing you can change on a live client — derive a new configuration with
`config.clone_with(...)` and build a new client instead.

```ruby
client = PaypalServerSdk::Client.new(
  environment: PaypalServerSdk::Environment::{Name},
  timeout: 30,
  max_retries: 3,
  retry_interval: 1,
  backoff_factor: 2,
  retry_statuses: [408, 429, 500, 502, 503, 504],
  retry_methods: %i[get put],
  proxy_settings: PaypalServerSdk::ProxySettings.new(address: 'http://localhost', port: 8888)
)
```

## Retries

The five retry arguments above are the complete set. **Their defaults are fixed when the SDK is
generated, not by the runtime** — `max_retries`, `retry_statuses` and `retry_methods` in particular differ
from SDK to SDK.

> Do not assume a default. Read the keyword-argument defaults on `Configuration#initialize` in
> **`lib/paypal_server_sdk/configuration.rb`** — that is the generated, authoritative value for this SDK,
> and `doc/client.md` restates each one in a **Default:** column. Retries may well be off
> (`max_retries: 0`).

Notes:

- `retry_methods` holds **symbols**, written with the `%i[...]` literal (`%i[get put]`), not strings.
- Only the methods in that list are retried. If `post`/`patch`/`delete` are absent — the common case —
  those failures surface with no retry. Add one only if the operation is idempotent.
- An operation the API marks as retriable is generated with `.endpoint_context('forced_retry', true)`
  and is retried **regardless of `retry_methods`**. Run `grep -rn forced_retry lib/paypal_server_sdk/` to
  see whether any operation you call does this — the controller files sit in a folder named after this
  SDK's controller namespace (`controllers/`, `apis/`, …), so do not assume the path.
- `retry_interval` is the pause in seconds before the first retry; `backoff_factor` multiplies it for
  each subsequent attempt.
- `retry_statuses` is a plain `Array` of integers.
- **Read the `max_retries:` default in `lib/paypal_server_sdk/configuration.rb`** — it is baked in when
  the SDK is generated, so it varies per build. At `0` nothing is retried and passing `max_retries: 0`
  is a no-op; above `0` every listed status and method is retried underneath your code whether you wanted it
  or not. Raise it only where nothing above the SDK already retries — a background job, a Sidekiq
  worker, a failover wrapper, or your own orchestration loop. Retry layers multiply rather than add:
  `max_retries: 3` is **four** requests per attempt, so inside a 3-attempt job it is twelve requests
  against an API whose rate limit counts every one.

## Timeout and transport

`timeout` is in **seconds** and is handed to Faraday, which applies it to the open, read *and* write
phases. Because retries happen inside the SDK, a call that retries can take substantially longer than
`timeout` — budget for `timeout × (max_retries + 1)` plus the backoff intervals when you set a deadline
of your own.

> ### ⚠ `timeout` does not bound a response that keeps trickling
>
> On the Net::HTTP-family adapters — read the `adapter:` default in `Configuration#initialize`, it is
> generated — the read timeout bounds **a single read**, not the whole response. A server that sends a
> byte before each deadline expires resets it every time, so the call can run indefinitely while
> `timeout` is set and working exactly as documented.
>
> This is the failure a stalled provider actually produces more often than a clean hang, and no setting
> on this client expresses it. If you need a true ceiling, impose a wall-clock deadline in your own code
> around the call.

**It is client-wide.** An operation's `options = {}` hash carries that operation's own parameters, not
request options: there is no per-call timeout, no per-call cancellation, and no way to give one call a
different budget than another. A call that needs one needs its own client — `config.clone_with(timeout:
5)` and a new client is the cheapest route, but read the `clone_with` caveat under **Retries** first.
Read the `timeout:` default in `Configuration#initialize` rather than assuming it; it is generated, and
a deadline you size from the wrong number is wrong in the direction that hurts.

> **A call that fetches a token spends `timeout` twice.** This applies to an SDK secured by an OAuth
> grant — check `lib/paypal_server_sdk/http/auth/` for an OAuth module; if there is none, skip this.
>
> An operation whose cached token is missing or expired fetches one first, as a **separate request**:
> the OAuth controller is built from the same configuration, so it shares the one Faraday connection and
> its `timeout`, subject to the trickle caveat above. That budget is per request, so the operation can
> take up to **two** full periods; size any caller-side deadline against two, not one.
>
> **The failure is disguised.** A failed fetch hands back the previously held token (`nil` on a first
> call) and what surfaces is the scheme's fixed `error_message` — the same message a wrong client id
> produces, with the underlying exception gone. To tell "the provider is down" from "our credentials are
> wrong", call `fetch_token` yourself at startup, or set an `o_auth_token_provider`.

Two lower-level arguments let you take over the transport entirely:

- **`adapter:`** — the Faraday adapter symbol to perform requests with.
- **`connection:`** — a `Faraday::Connection` you built yourself, used for every request. This is the
  hook for custom middleware, instrumentation or a stubbed transport (see **ruby-testing**).

## Proxy

```ruby
client = PaypalServerSdk::Client.new(
  proxy_settings: PaypalServerSdk::ProxySettings.new(
    address: 'http://localhost',
    port: 8888,
    username: 'user',
    password: 'pass'
  )
)
```

Only `address` is required. `ProxySettings.from_env` reads `PROXY_ADDRESS`, `PROXY_PORT`,
`PROXY_USERNAME` and `PROXY_PASSWORD`, returning `nil` when no address is set — `Client.from_env` calls
it for you.

## Base URL / environment

There is **no free-form base-URL argument**. The URL is looked up in the `ENVIRONMENTS` hash in
`lib/paypal_server_sdk/configuration.rb`, keyed by the environment you selected and then by the server each
endpoint chooses. Server template parameters declared by the spec become their own `Configuration`
arguments and are substituted by `get_base_uri`.

Both keys are frozen string constants on the `Environment` and `Server` classes in that
same file. **Read the `Environment` class for the real constant names before naming one** — they come from
this API's own server list, and a production member may not exist.

**A `connection:` you build with its own `url:` is ignored.** The client issues fully-resolved absolute
URLs, so pointing a Faraday connection at `http://localhost:4010` will not send requests there. Use
`connection:` to *intercept* requests instead (a Faraday test-adapter stub — see **ruby-testing** — or a
rewriting middleware), and `proxy_settings:` to route through a proxy.

**But the base URL itself is reachable, through the configuration object rather than through a
keyword.** The client wires its base-URI executor as a **bound method of the configuration you gave
it**:

```ruby
@global_configuration = GlobalConfiguration.new(client_configuration: @config)
                                           .base_uri_executor(@config.method(:get_base_uri))
```

and the constructor takes a ready-made one — `Client.new(config: my_config)`, in which case every other
keyword argument is ignored (see **ruby-getting-started**). So a `Configuration` subclass that overrides
`get_base_uri` retargets every request the SDK makes:

```ruby
class RetargetedConfiguration < PaypalServerSdk::Configuration
  def initialize(target_uri:, **kwargs)
    @target_uri = target_uri
    super(**kwargs)
  end

  def get_base_uri(server = nil)   # match the arity in your SDK's configuration.rb
    @target_uri
  end
end
```

Two properties are worth knowing before you rely on it:

- **It also retargets the token request**, because the auth managers are initialised from the same
  global configuration. That is usually what you want — retargeting only the API calls would leave the
  client authenticating against the original host — but it means a subclass that returns the new URL
  only for some servers has to account for the token endpoint deliberately.
- **It is a subclass of a generated class, so it is coupled to that class.** `get_base_uri`'s arity and
  the `Configuration` constructor's keywords are generated per SDK. Read
  `lib/paypal_server_sdk/configuration.rb` and match what is there, and treat a regeneration as something
  that can break this. If a proxy or DNS can get you to the same host, prefer it — those cost nothing
  when the SDK changes.

## Pagination
**No operation in this API is paginated**, so `lib/paypal_server_sdk/utilities/pagination/` is not generated
at all: there is no `PagedIterable`, no `PagedResponse` and nothing to call `.pages` or `.items` on. Every
list endpoint is a plain call returning the whole response the API sent.

Where such an endpoint takes paging parameters of its own (a `page`/`limit`/`cursor` the spec declares as
ordinary parameters), **you drive the loop**: pass the next value yourself and stop when a page comes back
with fewer items than you asked for, or when the API's own next-page field is empty. Nothing in the SDK
advances that state for you, and nothing bounds the loop — cap the iterations and the total items in your
own code.

## Logging

Logging is **built in**: this SDK ships it, so `Configuration#initialize` takes a
`logging_configuration:` argument and the classes live in `lib/paypal_server_sdk/logging/`. It is still
**off until you pass one** — the argument defaults to `nil`.

```ruby
client = PaypalServerSdk::Client.new(
  logging_configuration: PaypalServerSdk::LoggingConfiguration.new(
    log_level: Logger::INFO,
    mask_sensitive_headers: true,
    request_logging_config: PaypalServerSdk::RequestLoggingConfiguration.new(
      log_body: true,
      log_headers: true,
      headers_to_exclude: ['authorization'],
      include_query_in_path: true
    ),
    response_logging_config: PaypalServerSdk::ResponseLoggingConfiguration.new(
      log_body: true,
      log_headers: false
    )
  )
)
```

`log_body` and `log_headers` default to `false`, so a configuration you leave empty logs almost nothing.
Both request and response configurations also accept `headers_to_include` and `headers_to_unmask`, and
`LoggingConfiguration.from_env` reads `LOG_LEVEL`, `MASK_SENSITIVE_HEADERS` and the
`REQUEST_*`/`RESPONSE_*` variables.

Leave the logger argument unset to use the SDK's **built-in console logger**. To route logs elsewhere,
subclass `PaypalServerSdk::AbstractLogger` (`lib/paypal_server_sdk/logging/sdk_logger.rb`, documented in
`doc/abstract-logger.md`) and implement `log(level, message, params)` — `message` is a **template** and
`params` carries the values to interpolate. A stdlib `::Logger` does not satisfy that contract — it
swallows `params` and emits the template with its placeholders unsubstituted, with no error to tell
you.

`http_callback:` is still available alongside all of this, and is the seam to use when you want the
request/response objects themselves rather than log lines.

## Next

- Step 7, stubbing the SDK → **ruby-testing**
