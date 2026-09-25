---
name: 'ruby-getting-started'
description: 'Entry point for integrating the PayPal Server SDK Ruby SDK (APIMatic-generated, apimatic_core over Faraday). Carries the SDK''s identity — gem name, client class, environments, base exception — the package layout, and where to read the source. Load this first, before any other ruby-* skill.'
---

# Getting started with an APIMatic-generated Ruby SDK

This is the **SDK-specific** reference and grounding layer, and the entry into the companion `ruby-*`
skills. The SDK is built on the shared `apimatic_core_interfaces`, `apimatic_core` and
`apimatic_faraday_client_adapter` gems — the transport is
**Faraday**. For the general patterns that apply to *any* such SDK (client setup, auth, calling endpoints,
models, error handling, retries, testing), see the companion API-agnostic skills:
`ruby-client-initialization`, `ruby-authentication`, `ruby-calling-endpoints`, `ruby-models`,
`ruby-error-handling`, `ruby-configuration-resilience`, `ruby-testing`.

> **Before writing any integration code, locate the SDK source on disk** (see the *SDK source* section
> below) and read it to confirm every keyword argument, class, constant and exception type as you go. The
> source is obtained exactly as that section describes. Do **not** vendor or edit it inside your project;
> regenerate the SDK instead if something needs to change.

## SDK identity

| | |
| --- | --- |
| API | `PayPal Server SDK` |
| Runtime dependencies | `apimatic_core_interfaces`, `apimatic_core`, `apimatic_faraday_client_adapter` — read the `add_dependency` lines in `paypal_server_sdk.gemspec` for the pinned versions |
| Gem id | `paypal-server-sdk` (version `2.4.0`) — `s.name` / `s.version` in `paypal_server_sdk.gemspec` |
| Install | `gem install paypal-server-sdk` |
| Require | `require 'paypal_server_sdk'` — loads `lib/paypal_server_sdk.rb`, which requires everything else |
| Module | everything lives under `PaypalServerSdk` |
| Client | a single `PaypalServerSdk::Client` class, constructed with **keyword arguments**: `Client.new(timeout: 30, ...)` |
| Configuration | `PaypalServerSdk::Configuration < CoreLibrary::HttpClientConfiguration`; the client builds one from its own keyword arguments, or you pass a ready-made one as `config:` |
| Auth | credential objects passed as named parameters (e.g. `basic_auth_credentials:`) — this API declares at least one scheme, so `lib/paypal_server_sdk/http/auth/` exists and `Configuration#initialize` carries one credential parameter per scheme. See **ruby-authentication** |
| Environment | an `Environment` class of frozen string constants in `lib/paypal_server_sdk/configuration.rb` — read it for the real constant names. A generated SDK often has just one, so do not assume a production member exists |
| Controllers | **memoized reader methods on the client** — `client.{controller_name}` — not classes you instantiate |
| Return value | an `ApiResponse` — this SDK returns complete responses, so the deserialized value moves to `.data` and `status_code`, `reason_phrase`, `headers`, `raw_body` and `request` sit alongside it |
| Base error | `PaypalServerSdk::APIException < CoreLibrary::ApiException` — raised on error responses; see **ruby-error-handling** |
| Ruby version | `s.required_ruby_version` in `paypal_server_sdk.gemspec` (these SDKs target `>= 2.6`) |

Both were recorded with this SDK, so they are the coordinates a consumer installs rather than whatever this build happened to produce. Pin them as they stand.

## Package layout

An APIMatic Ruby SDK is a standard gem. `lib/paypal_server_sdk.rb` is the entry file — it `require`s every
other file, so a single `require 'paypal_server_sdk'` loads the whole SDK:

- `lib/paypal_server_sdk.rb` — the load manifest. Reading it is the fastest way to see exactly which files
  the SDK ships.
- `lib/paypal_server_sdk/client.rb` — the `Client` class: its keyword arguments, the memoized controller
  readers, `self.from_env`, and (for OAuth) the auth-manager readers.
- `lib/paypal_server_sdk/configuration.rb` — the `Environment` and `Server` constant classes, the
  `Configuration` class (**this is where every configurable keyword argument and its default lives**), the
  `ENVIRONMENTS` base-URL map, `get_base_uri`, and `self.build_default_config_from_env`.
- the **controller folder** under `lib/paypal_server_sdk/` — one controller class per API group. The
  folder name (`controllers/`, `apis/`, …) and the base class they extend (`BaseController`, `BaseApi`,
  …) are both named per SDK, the latter after its controller postfix, so do not
  assume a path:
  read the `# Controllers` block at the bottom of `lib/paypal_server_sdk.rb`, which `require_relative`s
  every controller file.
- `lib/paypal_server_sdk/models/` — one `BaseModel` subclass per model, plus enum classes. **This is where
  attribute names, optionality and enum values live.**
- `lib/paypal_server_sdk/exceptions/` — `api_exception.rb` (the `APIException` base) and one typed
  subclass per documented error response.
- `lib/paypal_server_sdk/http/` — `http_request.rb`, `http_response.rb`, `http_call_back.rb`,
  `http_method_enum.rb`, `proxy_settings.rb`, and `api_response.rb`, which is generated because this SDK returns complete responses.
  `http/auth/` holds the generated auth classes for the schemes this API declares.
- `lib/paypal_server_sdk/utilities/` — `date_time_helper.rb`, `file_wrapper.rb`, `union_type_lookup.rb`
  (when the API has oneOf/anyOf). There is no `pagination/` folder — no operation in this API is paginated.
- `lib/paypal_server_sdk/api_helper.rb` — `APIHelper < CoreLibrary::ApiHelper`.
- `lib/paypal_server_sdk/logging/` — the logging configuration classes, present because this SDK ships
  logging.

Most of these classes are thin subclasses of a `CoreLibrary::` class from the `apimatic_core` gem; the
generated file tells you the name, the gem holds the behaviour.

## Install

This SDK is published out of `https://github.com/paypal/PayPal-Ruby-Server-SDK`, branch `main` — take that branch explicitly rather than the repository default, which is not necessarily where this SDK is released from. Which registry that pipeline pushes to is a property of the pipeline rather than of the SDK. Try the install command below as-is first: if it resolves, the package is on the public registry and there is nothing further to configure. Only if it 404s do you need the feed — take it from the repository's publish workflow, or from whoever owns the pipeline, and configure that registry before retrying.

```bash
gem install paypal-server-sdk
```

In a Bundler project, add it to the `Gemfile` instead of installing it globally:

```ruby
gem 'paypal-server-sdk', '2.4.0'
```

That line still expects the gem to resolve from a source. For the common unpublished case — an SDK
handed over as a directory — point Bundler at the directory itself and skip the global install
entirely:

```ruby
gem 'paypal-server-sdk', path: 'vendor/paypal_server_sdk'
```

A rebuilt SDK carries a new version, and an already-installed gem stays selected until something asks
for the new one, so confirm what resolved:

```bash
gem list paypal-server-sdk --details
```

Then load it:

```ruby
require 'paypal_server_sdk'

client = PaypalServerSdk::Client.new
```

Every sample in these skills writes `PaypalServerSdk::` explicitly. `include PaypalServerSdk` also
works, but it pulls every SDK constant into the including scope.

## SDK source — read it in place

You will constantly need to confirm real keyword arguments, method signatures, model attributes, enum
constants and exception classes, and the **only reliable way** is to read the SDK source.

Clone it and read the clone — it is the source this pack documents:

```bash
git clone --filter=blob:none --branch main https://github.com/paypal/PayPal-Ruby-Server-SDK
```

**The clone is a branch; your install is pinned.** Check out the tag matching `2.4.0`, the version installed above before you read anything from it — a branch keeps moving after a release is cut, so the default checkout can be a different SDK than the one you compile against, and nothing in the tree will tell you. `git ls-remote --tags` lists what the repository offers; if no tag matches, treat every signature you read as unconfirmed rather than assuming it carried over. Clone it outside your project directory and treat it as read-only. It is a reference, not a dependency: what you build against is the package installed above, never this checkout.

An existing copy, if you already have one, is in whichever of these applies:

- the **unpacked SDK directory** you were given (the one containing the `.gemspec` and `lib/`); or
- the **installed gem** — `gem which paypal_server_sdk` prints the path of `lib/paypal_server_sdk.rb`, and
  `bundle show paypal-server-sdk` prints the gem root in a Bundler project.

Treat it as a read-only reference and grep it locally.

Layout — grep here first:

- `paypal_server_sdk.gemspec` — `s.name`, `s.version`, `s.required_ruby_version`, and the
  `add_dependency` lines.
- `lib/paypal_server_sdk/configuration.rb` — every configurable keyword argument **and its default**,
  the `Environment`/`Server` constants, the `ENVIRONMENTS` base-URL map, and the
  env-var names read by `build_default_config_from_env`.
- `lib/paypal_server_sdk/client.rb` — the `Client` keyword arguments and the controller reader names.
- the controller folder under `lib/paypal_server_sdk/` — its name is per SDK
  (`apis/`, `controllers/`, …); find it via the `# Controllers` block at the
  bottom of `lib/paypal_server_sdk.rb`, which `require_relative`s every controller file. Those files hold
  the operation methods and their parameter lists.
- `lib/paypal_server_sdk/models/` — model attributes, `self.names` (the Ruby-name-to-JSON-name map),
  `self.optionals`, `self.nullables`, and enum constants.
- `lib/paypal_server_sdk/exceptions/` — the typed exception classes and their attributes.
- `README.md` and `doc/` — a generated, human-readable index: `doc/client.md`,
  `doc/controllers/*.md` (one page per controller, with a copy-pasteable call and the operation's error
  table), `doc/models/*.md`, `doc/auth/*.md`, `doc/proxy-settings.md`, `doc/http-response.md`,
  `doc/http-request.md`, `doc/api-helper.md`, `doc/date-time-helper.md`,
  `doc/environment-based-client-initialization.md`. In the repository, it is the fastest way to find
  an operation, its parameters and its errors, then open the `.rb` file for the exact signature.
  **It is not in the installed gem** — a gem ships only what its gemspec's `files` list names, and
  the generated docs are not in it, so the root `bundle show paypal-server-sdk` prints holds the
  library and little else. With only the gem, `lib/paypal_server_sdk/controllers/` is the equivalent
  starting point.
- `bin/console` — a preconfigured IRB session that loads the SDK from `lib/`. Run `ruby bin/console` from
  the SDK root to poke at the client interactively.

## Placeholder legend

Samples across these skills write `{...}` for a name that differs per API. **They are never literal** —
resolve each one from this SDK's source before you write code. Anything already concrete for this SDK
(gem name, module, API name) is spelled out for you; if it is still in braces, it is yours to look up.

| Placeholder | Stands for | Resolve it from |
| --- | --- | --- |
| `{controller_name}`, `{controller}` | the memoized client reader a controller hangs off, as in `client.{controller_name}` | the reader definitions on the client class |
| `{operation}` | an operation method on a controller | the controller class body, or the operation list in `doc/controllers/`. The controller files live in the SDK controller folder — `controllers/` unless this SDK renamed the controller namespace |
| `{param}`, `{required_param}`, `{optional_param}`, `{other_param}`, `{status}` | one parameter of the operation you are calling | that operation's signature |
| `{Model}`, `{VariantModel}`, `{UnionName}` | a generated model class or union | the SDK's `models/` folder |
| `{attribute}`, `{required_attr}`, `{optional_attr}`, `{enum_attribute}`, `{union_attribute}`, `{list_attribute}`, `{map_attribute}`, `{message_attribute}`, `{error_attribute}` | one attribute of a model | that model's `initialize` keyword arguments |
| `{required_value}`, `{optional_value}`, `{wire_name}` | the value you pass for an attribute, and its wire name | your own data; the wire name is in the model's `names` mapping |
| `{EnumType}`, `{enum_type}`, `{ENUM_TYPE}`, `{SOME_CONSTANT}`, `{OTHER_CONSTANT}` | a generated enum and one of its constants | the SDK's `models/` folder |
| `{Name}` | an `Environment` constant | `configuration.rb` |
| `{ExceptionClass}`, `{OperationException}`, `{message}` | a typed exception class and its message attribute | the SDK's `exceptions/` folder |
| `{auth_key}` | the client reader for an auth manager, as in `client.{auth_key}` | the auth readers on the client class |
| `{Scheme}`, `{scheme}`, `{SCHEME}`, `{PARAM}` | an auth scheme and one of its parameters. `from_env` reads the bare `{PARAM}` when the API declares a **single** scheme and the prefixed `{SCHEME}_{PARAM}` when it declares several | `doc/auth/`, and the `from_env` method in the scheme's file under `http/auth/` |
| `{SchemeCredentials}`, `{BasicAuthCredentials}`, `{ApiKeyCredentials}`, `{OAuthCredentials}`, `{FirstCredentials}`, `{SecondCredentials}` | the credentials data class for one scheme | the scheme's file in the SDK's `http/auth/` folder — it holds the scheme class and, for every scheme but custom authentication, that scheme's credentials class |
| `{basic_auth_credentials}`, `{api_key_credentials}`, `{o_auth_credentials}`, `{first_credentials}`, `{second_credentials}` | the client keyword argument that carries those credentials | the client `initialize` signature |
| `{username_param}`, `{password_param}`, `{api_key_param}` | an individual field on a credentials class | that credentials class's `initialize` signature |
| `{placeholder}` | a stand-in value in an example | nothing to look up — substitute your own |

Any other `{...}` you meet is a local example; the sentence around it says what belongs there.

## Integration workflow — load the companion skill at each step

**Load the skill named for a step before you write that step's code, even where you have already read
the source.** The generated source is authoritative for the SDK's *surface*; these skills carry the
usage rules a signature cannot show, and each one names the trap that surface hides.

1. **ruby-client-initialization** — before you construct the client.
2. **ruby-authentication** — before you set credentials, and when a call returns 401 or 403.
3. **ruby-calling-endpoints** — before the first operation call.
4. **ruby-models** — as soon as a request or response field is not a plain string or number.
5. **ruby-error-handling** — before your first `begin`/`rescue` around a call.
6. **ruby-configuration-resilience** — before you ship. Its subject is the client's *defaults*, which apply whether or not you set anything, so this step is not conditional on changing one.
7. **ruby-testing** — before you stub the SDK.

## Next

- Step 1, construct the client → **ruby-client-initialization**

Every skill below ends with the step after it, so the footers walk the workflow above in order rather than offering a menu beside it.
