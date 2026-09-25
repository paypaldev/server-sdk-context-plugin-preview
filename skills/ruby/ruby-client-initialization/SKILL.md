---
name: 'ruby-client-initialization'
description: 'Construct and configure the PayPal Server SDK Ruby SDK client. Load before you call `Client.new` or wire the client into an application. The argument list won''t tell you settings are flat keyword arguments with no builder, that controllers are memoized readers, or how `clone_with` treats the arguments you leave out.'
---

# Initializing an APIMatic-generated Ruby SDK client

Names below are concrete for this SDK; replace the remaining `{...}` placeholders with the real names
from its source:

- `{controller_name}` — a controller reader method on the client.
- `{Name}` — an `Environment` constant.

## The shape: keyword arguments, no builder and no options hash

The SDK exports a single `Client` class under `PaypalServerSdk`. You construct it with **keyword
arguments** — a flat list, not an options object and not a builder. Every argument has a default that was
fixed when the SDK was generated, so passing none is valid:

```ruby
require 'paypal_server_sdk'

client = PaypalServerSdk::Client.new(
  environment: PaypalServerSdk::Environment::{Name},
  timeout: 30,
  # auth credential objects — see ruby-authentication
  # retry / proxy / callback arguments — see ruby-configuration-resilience
)
```

`Client#initialize` declares the **same** arguments as `Configuration#initialize` plus a `config:`, and
simply forwards them into a new `Configuration`. Open
`lib/paypal_server_sdk/configuration.rb` for the authoritative list and the real defaults — the set varies
per API. In this build that set includes one credential argument per declared scheme, any server template
parameters the API declares, and a `logging_configuration:`.

The arguments common to every build:

| Argument | Type | Purpose |
| --- | --- | --- |
| `connection` | `Faraday::Connection` | a Faraday connection you built yourself, used for every request |
| `adapter` | `Faraday::Adapter` | the Faraday adapter to perform requests with |
| `timeout` | `Float` | connection timeout in **seconds** |
| `max_retries` | `Integer` | how many times a failed call is retried |
| `retry_interval` | `Float` | pause in seconds between retries |
| `backoff_factor` | `Float` | multiplier applied to each successive retry interval |
| `retry_statuses` | `Array` | HTTP status codes that are retried |
| `retry_methods` | `Array` | HTTP methods that are retried, as symbols |
| `http_callback` | `HttpCallBack` | pre/post API call hooks |
| `proxy_settings` | `ProxySettings` | route requests through a proxy |
| `environment` | `Environment` | which environment's base URL to use |
| `config` | `Configuration` | a ready-made configuration — see the warning below |

> **`config:` overrides everything.** `Client#initialize` builds a `Configuration` from the other keyword
> arguments **only when `config` is nil**. Pass a `config:` and every other argument you passed alongside
> it is silently ignored. Use one form or the other, never both.

## Choosing the environment / base URL
Environments are frozen string constants on an `Environment` class in
`lib/paypal_server_sdk/configuration.rb`. **Read that class for the real constant names before naming
one.**

Do not assume a particular name exists — there may be
no production member at all. A name also does **not** imply a live host: match each constant to the URL it
actually resolves to in the `ENVIRONMENTS` hash in the same file.

```ruby
client = PaypalServerSdk::Client.new(environment: PaypalServerSdk::Environment::{Name})
```

`Configuration#get_base_uri(server)` looks the URL up in the `ENVIRONMENTS` hash, keyed by the selected environment and then by a `Server` constant (an API with
multiple servers generates several; endpoints pick their own). When the spec declares server template
parameters, those become their own `Configuration` arguments and are substituted into the URL by
`get_base_uri` — grep that method to see whether this SDK has any.

`Environment.from_value` accepts a string or symbol, and is how `ENVIRONMENT` from the process
environment gets resolved.

> **Its fallback is the *first declared* environment, not the SDK's default one.** `from_value`'s
> `default_value` parameter is generated as the first constant in the spec's environment list, so a
> `nil` or unrecognised value resolves there — **and the two are the same constant only when the API
> definition happens to list its default first.** An unrecognised value also only warns; it does not
> raise. So a typo in `ENVIRONMENT`, or the variable being unset, silently selects the first
> environment, which may be the live one. Read the `def self.from_value` line in
> `lib/paypal_server_sdk/configuration.rb` to see which constant that actually is, and pass
> `environment:` explicitly rather than relying on the fallback.
>
> The same generation rule applies to `Server.from_value`, whose fallback is the **first** environment's
> first server — not the selected environment's.

## Configuration from environment variables

`Client.from_env` builds a client without writing the arguments by hand:

```ruby
require 'paypal_server_sdk'

client = PaypalServerSdk::Client.from_env
```

It calls `Configuration.build_default_config_from_env`, which reads a fixed set of `UPPER_SNAKE`
variables — `ENVIRONMENT`, `TIMEOUT`, `MAX_RETRIES`, `RETRY_INTERVAL`, `BACKOFF_FACTOR`, `RETRY_STATUSES`,
`RETRY_METHODS`, the `PROXY_*` set, the logging set (`LOG_LEVEL`, `MASK_SENSITIVE_HEADERS`, the `REQUEST_*` and
`RESPONSE_*` variables), plus one per auth credential and per server parameter. **Grep `build_default_config_from_env` in
`lib/paypal_server_sdk/configuration.rb` for the exact names this SDK reads** — the auth ones are generated
per scheme, and `doc/environment-based-client-initialization.md` lists the whole set for this build.
Anything unset falls back to the SDK's own default.

> **The logging variables have a side effect.** `build_default_config_from_env` builds a logging
> configuration whenever *any* of them is set, so exporting a single `LOG_LEVEL` or `REQUEST_LOG_BODY`
> turns request/response logging on for a `from_env` client even though you passed no
> `logging_configuration:`.

`from_env` also takes overrides, which win over the environment:

```ruby
client = PaypalServerSdk::Client.from_env(environment: PaypalServerSdk::Environment::{Name}, timeout: 30)
```

To load a `.env` file first, add the `dotenv` gem and `require 'dotenv/load'` **before** you call
`from_env`. `dotenv` is not a dependency of the SDK — you add it yourself.

## Accessing controllers — memoized readers on the client

Unlike some SDKs, you do **not** instantiate controllers. Each is a reader method on the client that
builds its controller once and memoizes it:

```ruby
result = client.{controller_name}.{operation}
```

Read the reader names off `lib/paypal_server_sdk/client.rb` (or the controller/API table near the bottom
of `doc/client.md` — it is headed `## Apis` or `## Controllers` after the controller folder, which is
named per SDK) — they are snake_cased from the API's group names,
so don't guess them. See
**ruby-calling-endpoints** for the call itself.

Those readers are the only ones on the client, with one exception: an SDK whose API uses **OAuth** also
exposes an auth manager reader per scheme (named after the scheme, e.g. `client.{auth_key}`) plus an
`auth_managers` hash — used to fetch and refresh tokens. See **ruby-authentication**.

## Client lifetime and reuse

`Client` builds its `Configuration`, its global configuration and its Faraday client **once** in the
constructor, and `Configuration` exposes readers only. Treat the client as **immutable and long-lived**:
construct it once at startup and reuse it for the process lifetime. Do not build a new client per request
— that discards connection pooling and any cached OAuth token.

```ruby
# startup — construct once:
API_CLIENT = PaypalServerSdk::Client.new(environment: PaypalServerSdk::Environment::{Name})

# elsewhere — reuse:
API_CLIENT.{controller_name}.{operation}
```

To produce a variant with a few settings changed, derive a new configuration and build a new client from
it — this is also how a freshly fetched OAuth token gets attached:

```ruby
new_config = client.config.clone_with(timeout: 60)
client = PaypalServerSdk::Client.new(config: new_config)
```

`clone_with` returns a **new** `Configuration` carrying the current values for everything you don't pass;
it never mutates the original. Note it fills a missing argument with `||=`, so an argument you pass as
**`false` or `nil` is discarded** and the existing value survives — those are Ruby's only falsy values.
`0` is truthy in Ruby, so `clone_with(max_retries: 0)` *does* take effect; it is the boolean settings
that cannot be cleared this way. Build a fresh `Configuration` where you need to force one to `false`.

## Next

- Configure authentication → **ruby-authentication**
- Make your first call → **ruby-calling-endpoints**
- Tune retries/timeouts/proxy → **ruby-configuration-resilience**
