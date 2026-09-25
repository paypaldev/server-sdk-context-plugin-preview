---
name: 'ruby-calling-endpoints'
description: 'Call operations on the PayPal Server SDK Ruby SDK. Load before the first call, and when building a request body or reading a response. The method won''t tell you the controller folder is named per SDK, whether parameters come positionally or as a hash, or what an operation returns on an error status.'
---

# Calling endpoints on an APIMatic Ruby SDK

> **`doc/` is in the SDK's repository, not in the installed package.** Every pointer to
> `doc/controllers/` or `doc/client.md` below assumes you have the repository checked out. Where you
> only have the installed artifact — the common case — the equivalent source is the operation's own `def` line in `lib/paypal_server_sdk/controllers/`, and it is
> authoritative in a way the docs are not.

Operations are methods on a **controller you reach through a reader on the client**. There is no
controller class to instantiate:

```ruby
require 'paypal_server_sdk'

client = PaypalServerSdk::Client.new
result = client.{controller_name}.{operation}
```

Each reader (`client.{controller_name}`) builds its controller once and memoizes it, so calling it
repeatedly is free. Controllers live in a folder under `lib/paypal_server_sdk/` — one file per API group.
The folder name (`controllers/`, `apis/`, …) and the base class they extend (`BaseController`, `BaseApi`,
…) are both named per SDK — the latter after the controller postfix — so read the
`# Controllers` require block at the bottom of `lib/paypal_server_sdk.rb` for the real folder instead of
assuming one. **Read the reader names off `lib/paypal_server_sdk/client.rb`** (or the controller/API table
in `doc/client.md`); operation method names are snake_cased from the spec and follow no fixed
verb/resource pattern, so take the real name from the source too.

## Method signature convention

**Which of three parameter forms an operation uses is decided when the SDK is generated, so read its
`def` line before writing the call.** The unset form is positional required parameters with keyword
optionals:

```ruby
def {operation}({required_param},
                {optional_param}: nil,
                _query_parameters: nil)
```

- **Required parameters are positional here**, in the order the spec declares them.
- **Optional parameters are keyword arguments**, so you pass only the ones you need and never a
  positional placeholder. They usually default to `nil`, but a parameter the spec gives a default gets
  that value — and it is **sent on the wire**, so omitting the argument does not mean the parameter is
  omitted from the request. Read the `def` line.
- **Some SDKs make every parameter a keyword argument** — where an SDK is built that
  way, a required parameter reads `{required_param}:` instead. A collecting SDK (or a single endpoint
  that collects on its own) instead gathers every parameter of an operation that has more
  than one into a single hash, `def {operation}(options = {})`, leaving no positional required parameters
  and no per-parameter keyword optionals — only the `_query_parameters:` / `_field_parameters:` escape
  hatches below can still trail the hash. **That hash is keyed by strings**, not symbols and not keyword
  arguments — the generated body reads each value back as `options['{param}']`, and a request body is
  `options['body']`. Writing `{param}: value` parses fine, builds a symbol-keyed hash, and every lookup
  returns `nil`: the request goes out with its template parameter unsubstituted and no query values,
  with **no Ruby-level error**. Write it as `{operation}('{param}' => value, ...)`, taking the names from
  the `@param` comments above the `def`. An operation that declares exactly one parameter stays
  positional even in a collapsed build — but that is a count of **declared** parameters, not of required
  ones, so one required parameter followed by a few optional ones is a multi-parameter operation and does
  collect into the hash. **Read the `def` line** in the controller file (or the signature block in
  `doc/controllers/{controller}.md`) before writing the call — do not infer it from another operation.
- **A nullable parameter is a keyword argument even when it is required**, because `nil` has to be
  expressible.
- **`_query_parameters:` / `_field_parameters:`** appear on operations that accept extra, unmodelled
  query or form values. Each takes a `Hash` that is merged into the request — an escape hatch for
  parameters the spec doesn't name, not something you normally set.
- Methods **raise** on error responses — see **ruby-error-handling**.

## Reading the response



Every **non-paginated** operation in this SDK returns an
`ApiResponse`, with the deserialized payload on `.data`. That applies to the whole SDK at once —
there is no per-operation exception:

```ruby
response = client.{controller_name}.{operation}

if response.success?
  puts response.data
elsif response.error?
  warn response.errors
end
```

`ApiResponse` carries `status_code`, `reason_phrase`, `headers`, `raw_body`, `request` and `data`. The
payload is on **`.data`** — not `.body` and not `.result` — so a sample written against a bare-value SDK
(`pets.each { ... }` straight off the call) iterates the wrapper and fails. You can confirm the build in
one glance: `lib/paypal_server_sdk/http/api_response.rb` exists only in an SDK that returns the wrapper.

**Binary responses have no model.** An operation whose response handler declares neither a
`deserialize_into` nor `is_response_void(true)` hands you the raw body — `doc/controllers/{controller}.md`
types it `Binary`. Read the bytes off `response.data` (or `response.raw_body`) and write them out
yourself. A handler marked `is_response_void(true)` is the empty-response case instead — but do not expect `nil`
from it: with no `deserializer` registered, what reaches `data` is the **raw body String**, empty when
the server sent no body. Test it with `.empty?` rather than `.nil?`, and check for that qualifier before
concluding a response is binary.



## Building request models

A request body parameter is a model instance from `lib/paypal_server_sdk/models/`. Construct it and pass
it like any other parameter:

```ruby
# keyword form (EnableModelKeywordArgsInRuby)
body = PaypalServerSdk::{Model}.new({required_attr}: {required_value}, {optional_attr}: {optional_value})

# positional form
body = PaypalServerSdk::{Model}.new({required_value}, {optional_value})

result = client.{controller_name}.{operation}(body)

# collapsed signature (def {operation}(options = {})) — the body travels under the string key 'body'
result = client.{controller_name}.{operation}('body' => body)
```

**Open the model's `initialize`** and use the form it declares — `EnableModelKeywordArgsInRuby` decides
which one this SDK has, and calling the wrong one raises `ArgumentError: wrong number of
arguments`. The SDK's own `doc/models/` examples use whichever form this build has. Attributes you leave
out are omitted from the serialized JSON entirely. See **ruby-models** for the details, and for anything that isn't a plain
string or number (enums, oneOf/anyOf unions, collections, dates, file uploads).

## Enums

An enum is a class of constants, not a Ruby symbol — frozen strings, or plain `Integer`s when the spec
declares an integer-based enum. Pass the constant; passing a raw value directly is safe only for a string
enum, since an integer enum's constant holds a number and a string would go on the wire as one:

```ruby
client.{controller_name}.{operation}(PaypalServerSdk::{EnumType}::{SOME_CONSTANT})
client.{controller_name}.{operation}('server_provided_value')   # string enums only
```

The constants live in `lib/paypal_server_sdk/models/{enum_type}.rb`. See **ruby-models**.

## Worked example — a list call with optional filters

```ruby
# Signature (illustrative — read the real one):
#   def {operation}({status},
#                   page: nil,
#                   per_page: nil)

result = client.{controller_name}.{operation}(
  PaypalServerSdk::{EnumType}::{SOME_CONSTANT},
  page: 1,
  per_page: 100
)

result.data.each do |item|
  puts item.{attribute}
end
```

The same call against a collapsed signature — `def {operation}(options = {})` — is one string-keyed hash,
required and optional parameters together:

```ruby
result = client.{controller_name}.{operation}(
  '{status}' => PaypalServerSdk::{EnumType}::{SOME_CONSTANT},
  'page' => 1,
  'per_page' => 100
)
```

## Finding the right method in the SDK source

- `doc/controllers/` — one markdown page per controller: the reader used to obtain it, the class name,
  every operation's `def` line, its response type, a copy-pasteable example, and an **Errors table**
  mapping status codes to exception classes. **Grep here first.**
- the controller files under `lib/paypal_server_sdk/` (folder named per the `# Controllers` require block
  in `lib/paypal_server_sdk.rb`) — the authoritative signatures, plus the request builder that shows the
  HTTP method, the route and how each parameter is sent (path, query, header, form, body).
- `lib/paypal_server_sdk/models/` — request/response models and enums.
- `lib/paypal_server_sdk/exceptions/` — the exception classes named in the Errors tables.

## Next

- Unions, collections, dates, enums → **ruby-models**
- Errors and status codes → **ruby-error-handling**
- Pagination, retries, timeouts → **ruby-configuration-resilience**
