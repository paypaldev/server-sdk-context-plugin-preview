---
name: 'php-client-initialization'
description: 'Construct and configure the PayPal Server SDK PHP SDK client. Load before you call the builder or wire the client into an application. The method list won''t tell you the client is immutable and built through `{Client}Builder::init()`, that controllers are accessors whose suffix is per-SDK, or which settings a rebuilt client keeps.'
---

# Initializing an APIMatic-generated PHP SDK client

Replace the `{...}` placeholders below with the real names from your SDK's source:

- `{Client}` — the client class in `src/`, e.g. `src/AcmeClient.php`. Its builder is `{Client}Builder`.
- `{Group}` — an API group; the client exposes one memoized accessor per group, `get{Group}{Postfix}()`.
- `{Postfix}` — the accessor/class suffix, which is named per SDK. It is
  `Controller` unless this one renamed it (`Api`, `Client`, …), and that one name covers the folder, the
  classes and the client getters together — `ls src/` and read the accessor names off `src/{Client}.php` before
  writing one.

An API split across multiple namespaces generates **one client class per namespace**, so `ls src/*.php`
rather than assuming a single client.

Substitute these before running anything: `use PaypalServerSdkLib\{Client}Builder;` left as written is a
PHP **parse error**, since `\{…}` after a namespace is group-use syntax rather than an identifier.

## The shape: a builder, not a constructor

```php
use PaypalServerSdkLib\{Client}Builder;
use PaypalServerSdkLib\Environment;

$client = {Client}Builder::init()
    ->environment(Environment::{CONSTANT})
    ->timeout(30)                     // SECONDS, not milliseconds
    // credentials — see php-authentication
    ->build();
```

`{Client}Builder::init()` is a static factory (the constructor is private). Every setter returns `$this`,
and `build()` hands the accumulated config array to the client.

**The builder's setter list is the complete, authoritative set of supported options.** If an option is
not a method on `src/{Client}Builder.php`, this SDK does not support it — there is no options array, no
escape hatch and no way to reach the underlying HTTP client. Open that file before assuming anything is
configurable.

The client also has a public `__construct(array $config = [])` taking the raw config array, but its
docblock points at the builder: the keys are internal and unchecked, so use the builder.

### Options every generated SDK carries

| Setter | Type | Notes |
| --- | --- | --- |
| `timeout` | `int` | **seconds** |
| `enableRetries` | `bool` | master switch for the retry feature |
| `numberOfRetries` | `int` | |
| `retryInterval` | `float` | seconds |
| `backOffFactor` | `float` | exponential multiplier |
| `maximumRetryWaitTime` | `int` | seconds — a cumulative budget for the backoff waits, not a cap; too small and nothing retries |
| `retryOnTimeout` | `bool` | |
| `httpStatusCodesToRetry` | `int[]` | |
| `httpMethodsToRetry` | `string[]` | |
| `environment` | `string` | pass an `Environment::` constant |
| `httpCallback` | *(untyped)* | pre/post request hook; must be a `CoreCallback` — see the warning below |
| `proxyConfiguration` | `ProxyConfigurationBuilder` | |

Retry semantics and defaults are in **php-configuration-resilience** — do not guess them.

Additional setters appear **conditionally**, so their absence is meaningful:

- one `->{scheme}Credentials(…)` per auth scheme — this API declares at least one, so
  expect them (see **php-authentication**);
- one per server parameter declared by the API;
- `loggingConfiguration(…)`, which this SDK has;
- `additionalHeaders(…)` / `userAgentDetail(…)` / `skipSslVerification(…)`, each present only in an SDK
  built with it — read the builder to see which of them yours has.

> **`httpCallback()` fails silently.** Its implementation is
> `if (!$httpCallback instanceof CoreCallback) { return $this; }` — pass anything else and the call is
> a no-op with no error. Pass the SDK's own `PaypalServerSdkLib\Http\HttpCallBack` (or another
> `CoreCallback` subclass).

## Choosing the environment / base URL

Environments are `public const` strings on `class Environment` in `src/Environment.php` — a plain class,
not a PHP `enum`. **Read that file for the real constant names before naming one.**

```php
$client = {Client}Builder::init()
    ->environment(Environment::{CONSTANT})
    ->build();
```

Do not assume a particular name exists — there may be no
`PRODUCTION` at all. A name also does not imply a live host: match each constant to the URL it actually
resolves to in the `ENVIRONMENT_MAP` private const in `src/{Client}.php`.

`environment()` takes a `string`, so nothing stops you passing an arbitrary value — but the base URL is
looked up in that private map by exactly this key, so an unknown environment has no URL to resolve to.
Some APIs expose server parameters (a templated host segment,
a port) as their own builder setters that feed the URL template; check `src/{Client}Builder.php`.

To confirm what a built client actually resolved to:

```php
echo $client->getBaseUri();                  // default server
echo $client->getBaseUri(Server::{ALIAS});   // a named server alias, from src/Server.php
```

## Accessing controllers — accessors on the client

Unlike some SDKs, you do **not** construct controllers. The client exposes one accessor per API group
and memoizes the instance:

```php
$controller = $client->get{Group}{Postfix}();   // e.g. getShipmentsController(), or getShipmentsApi()
                                                // in a renamed build — read src/{Client}.php
$result = $controller->{operation}(/* … */);
```

Read the real accessor names off the accessor block at the end of `src/{Client}.php` (each method returns
a group class and memoizes it), or off the group table in `doc/client.md`, whose heading is the pluralized
suffix (`## Apis` or `## Controllers`). Both the group name and the suffix come from generation — do not
assume either. See **php-calling-endpoints** for the call itself.

## Client lifetime, immutability and reuse

The client merges the builder's config over `ConfigurationDefaults::_ALL` **once** in its constructor and
builds its internal HTTP client there. Treat it as **immutable and long-lived**: construct it once at
startup and reuse it for the process lifetime. Building a client per request throws away connection
reuse and any cached OAuth token.

There is no setter on a built client. To produce a variant:

```php
// Preferred — round-trip through the builder, so you use the same typed setters:
$other = $client->toBuilder()
    ->timeout(5)
    ->build();

// Or merge raw config keys (internal key names — prefer toBuilder()):
$other = $client->withConfiguration(['timeout' => 5]);
```

Both return a **new** client; the original is untouched. `toBuilder()` is also how you attach a
refreshed OAuth token — see **php-authentication**.

`$client->getConfiguration()` returns the current config as an array (useful for logging what a client
was actually built with).

## Next

- Configure authentication → **php-authentication**
- Make your first call → **php-calling-endpoints**
- Tune retries/timeouts/proxy/logging → **php-configuration-resilience**
