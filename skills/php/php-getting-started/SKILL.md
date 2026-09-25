---
name: 'php-getting-started'
description: 'Entry point for integrating the PayPal Server SDK PHP SDK (APIMatic-generated, apimatic/core over apimatic/unirest-php). Carries the SDK''s identity — Composer package and root namespace, client builder, environments, base exception — the package layout, and where to read the source. Load this first, before any other php-* skill.'
---

# Getting started with an APIMatic-generated PHP SDK

This is the **SDK-specific** reference and grounding layer, and the entry into the companion `php-*`
skills. The SDK is built on the shared `apimatic/core` + `apimatic/core-interfaces` runtime over
the `apimatic/unirest-php` HTTP client. For
the general patterns that apply to *any* such SDK (client setup, auth, calling endpoints, models, error
handling, resilience, testing), see the companion skills: `php-client-initialization`,
`php-authentication`, `php-calling-endpoints`, `php-models`, `php-error-handling`,
`php-configuration-resilience`, `php-testing`.

> **Before writing any integration code, locate the SDK source on disk** (see the *SDK source* section
> below) and read it to confirm every class name, method signature, model field and exception type as
> you go. The source is obtained exactly as that section describes. Do **not** vendor or edit it inside your
> project; regenerate the SDK instead if something needs to change.

## SDK identity

| | |
| --- | --- |
| API | `PayPal Server SDK` |
| Runtime dependencies | `apimatic/core`, `apimatic/core-interfaces`, `apimatic/unirest-php`, plus the `json` and `curl` PHP extensions — see `composer.json` `require` |
| Composer package | `paypal/paypal-server-sdk` (SDK version `2.4.0`) — the `"name"` in `composer.json` |
| Install | `composer require paypal/paypal-server-sdk` |
| PSR-4 root namespace | `PaypalServerSdkLib` → `src/` (`composer.json` `autoload.psr-4`) — **this, not the package name, is what `use` statements reference** |
| PHP version | per `composer.json` `require.php` (these SDKs target `^7.2 \|\| ^8.0`) |
| Client | one `{Client}` class per API namespace, built with `{Client}Builder::init()->…->build()` |
| Configuration | fluent setter methods on `{Client}Builder`; every default lives in `src/ConfigurationDefaults.php` |
| Auth | **this API declares at least one scheme**: `->{scheme}Credentials({Scheme}CredentialsBuilder::init(…))` on the client builder, with the builders and managers under `src/Authentication/`. Read the real setter names off `src/{Client}Builder.php` — a single-scheme API is named after the auth *type*, not after the spec's scheme. See **php-authentication** |
| Environment | `class Environment` of `public const` strings in `src/Environment.php` — read it for the real constant names |
| Controllers | accessors on the client: `$client->get{Group}{Postfix}()` — the postfix is named per SDK, `Controller` unless this one renamed it, so `getShipmentsController()` or `getShipmentsApi()`; read `src/{Client}.php` for the real accessor names — **not** classes you instantiate |
| Return type | an `ApiResponse` wrapper (`getResult()`, `getStatusCode()`, `getHeaders()`, `isSuccess()`) on **every** operation — `src/Http/ApiResponse.php` and `doc/api-response.md` are generated, and nothing here returns the bare value. Confirm on the `@return` of any method in the controller directory. |
| Base error | `ls src/Exceptions/` — one class per error response this API documents, plus the provider's own error class on OAuth-grant builds, and this build generates at least one. Whether a shared `ApiException` base exists, and whether these classes are `\Throwable` at all, varies per SDK; when error types are mapped into the `ApiResponse` the SDK throws nothing and they are plain models. Note the return shape decides where a bad *status* surfaces: this SDK returns the `ApiResponse` wrapper, so a non-2xx comes back inside it and never raises. See **php-error-handling**. |

Both were recorded with this SDK, so they are the coordinates a consumer installs rather than whatever this build happened to produce. Pin them as they stand.

## Package layout

Everything lives under `src/`, mapped to `PaypalServerSdkLib\` by PSR-4. There is no barrel file — you
`use` each fully-qualified class:

- `src/{Client}.php` — the client. Holds the merged config array, exposes the controller accessors, the
  config getters (`getTimeout()`, `getEnvironment()`, …), `getBaseUri($server)`, `toBuilder()` and
  `withConfiguration(array $config)`. The environment → base-URL map is a **private const** here.
- `src/{Client}Builder.php` — `init()`, one fluent setter per configurable option, `build()`.
- `src/ConfigurationDefaults.php` — the generated defaults (`TIMEOUT`, `NUMBER_OF_RETRIES`,
  `HTTP_METHODS_TO_RETRY`, …). **This is the authoritative default for this SDK**, not any documented
  norm; the values are fixed when the SDK is generated.
- `src/ConfigurationInterface.php` — what the client guarantees to expose. That includes
  one credentials accessor plus one `get{Scheme}CredentialsBuilder()` per auth scheme. The credentials
  accessor is `get{Scheme}Credentials()`, **except** when the API has exactly one scheme and it is an
  OAuth 2 grant, where the `Credentials` suffix is dropped and it is `get{Scheme}()`. Read the file; a grep
  for the suffixed name alone can wrongly look like the scheme is absent.
- `src/Environment.php` and `src/Server.php` — the environment constants and the base-URL aliases.
- the controller directory — `src/Controllers/` by default, or another name (e.g. `src/Apis/`) in an SDK
  that renamed the controller postfix or its namespace; note the directory is
  the **pluralized** postfix (`Api` → `Apis`). One `{Group}{Postfix}` class per API group, each extending
  `Base{Postfix}` — `ShipmentsController extends BaseController` by default, `ShipmentsApi extends
  BaseApi` in a renamed build. `ls src/` to see which this SDK has.
- `src/Models/` — model classes (private fields + `get`/`set` pairs + `jsonSerialize()`), with
  `src/Models/Builders/{Model}Builder.php` beside them. **This is where field names, required-vs-optional
  and enum values live.**
- `src/Exceptions/` — one class per documented error response; check each file's `class ... extends ...`
  line before catching it.
- `src/Http/` — `HttpRequest`, `HttpResponse`, `HttpContext`, `HttpMethod`, `HttpCallBack`, and
  `ApiResponse`, which this SDK generates; thin subclasses
  of the `apimatic/core` types.
- `src/Proxy/ProxyConfigurationBuilder.php` — proxy settings.
- `src/ApiHelper.php` — `serialize()` / `deserialize()` / `stringify()` and the JsonMapper instance.
- `src/Authentication/` — **generated**: this API declares at least one auth scheme, so
  the credentials builders and managers live here. See **php-authentication**.
- `src/Logging/` — **generated**: this SDK has logging built in, so the
  logging configuration builders are here and `loggingConfiguration(…)` is on the client
  builder. See **php-configuration-resilience**.
- `src/Utils/DateTimeHelper.php` and `src/Utils/FileWrapper.php` — present **only** when the API has
  date or file types respectively. Their absence is meaningful: don't assume a class exists, `ls` the
  directory.

## Install

This SDK is published out of `https://github.com/paypal/PayPal-PHP-Server-SDK`, branch `main` — take that branch explicitly rather than the repository default, which is not necessarily where this SDK is released from. Which registry that pipeline pushes to is a property of the pipeline rather than of the SDK. Try the install command below as-is first: if it resolves, the package is on the public registry and there is nothing further to configure. Only if it 404s do you need the feed — take it from the repository's publish workflow, or from whoever owns the pipeline, and configure that registry before retrying.

`paypal/paypal-server-sdk` is the Composer package name; `PaypalServerSdkLib` is the PHP namespace. They are
different strings and both matter.

The install command for this SDK is:

```bash
composer require paypal/paypal-server-sdk
```

If the command above is a bare `composer require` and Composer answers `Could not find a matching
version of package paypal/paypal-server-sdk`, the SDK is **not on a registry** — you were handed an unpacked
directory, which is the common case, and Composer reaches it through a **path repository** instead. The
steps below are what the unpublished form of that command condenses into one line — register the
directory as a path repository and require it at `:@dev`, since a local checkout carries no version tag.

Confirm what resolved:

```bash
composer show paypal/paypal-server-sdk
```

Once the SDK *is* published it is pinned in `composer.json` like any other dependency — regenerating
publishes a new version, so pin rather than float:

```json
"require": {
    "paypal/paypal-server-sdk": "2.4.0"
}
```

Then load Composer's autoloader and import by namespace:

```php
require_once 'vendor/autoload.php';

use PaypalServerSdkLib\{Client}Builder;
use PaypalServerSdkLib\Environment;
```

`{Client}` is a **placeholder**, here and in every companion skill — substitute the real class name (the
one `src/*ClientBuilder.php` file). Left as written it is a PHP **parse error**, because `\{…}` after a
namespace is group-use syntax (`use Foo\{A, B};`), not an identifier.

### Two environment prerequisites that fail silently or confusingly

- **`opcache.save_comments` must stay enabled** (it is `1` by default). Deserialization is driven by
  JsonMapper, which reads the `@var`, `@maps` and `@factory` docblock annotations at runtime. With
  comments stripped, models come back unpopulated rather than erroring. Check with
  `php -i | grep opcache.save_comments`.
- **On Windows, cURL ships without a CA bundle.** An HTTPS call then fails with
  `SSL certificate problem: unable to get local issuer certificate`. Download `cacert.pem` from
  <https://curl.haxx.se/docs/caextract.html> and point `php.ini` at it with an absolute path:
  `[curl]` / `curl.cainfo = "C:\absolute\path\to\cacert.pem"`.

## SDK source — read it in place

You will constantly need to confirm real class names, method signatures, model fields and exception
types, and the **only reliable way** is to read the SDK source.

Clone it and read the clone — it is the source this pack documents:

```bash
git clone --filter=blob:none --branch main https://github.com/paypal/PayPal-PHP-Server-SDK
```

**The clone is a branch; your install is pinned.** Check out the tag matching `2.4.0`, the version installed above before you read anything from it — a branch keeps moving after a release is cut, so the default checkout can be a different SDK than the one you compile against, and nothing in the tree will tell you. `git ls-remote --tags` lists what the repository offers; if no tag matches, treat every signature you read as unconfirmed rather than assuming it carried over. Clone it outside your project directory and treat it as read-only. It is a reference, not a dependency: what you build against is the package installed above, never this checkout.

An existing copy, if you already have one, is in whichever of these applies:

- the **unpacked SDK directory** you were given (the one containing `composer.json` and `src/`); or
- **`vendor/paypal/paypal-server-sdk/`** in your project, once installed.

Treat it as a read-only reference and grep it locally.

Grep here first:

- `composer.json` — the package `name`, the `require` block, and `autoload.psr-4` (the root namespace).
- `src/{Client}Builder.php` — the complete, authoritative list of configurable options. If a setter is
  not on this class, the SDK does not support it.
- `src/ConfigurationDefaults.php` — the real default for every one of those options.
- `src/Environment.php` — the environment constants; `src/{Client}.php` for the base URL each maps to.
- the controller directory (`src/Controllers/`, `src/Apis/`, … — `ls src/` first) — operation signatures
  and parameter order.
- `src/Models/` — field names, required-vs-optional, and `src/Models/Builders/` for how to construct them.
- `src/Exceptions/` — which typed exceptions exist.
- `README.md` and `doc/` — a generated, human-readable index: `doc/client.md` (the full configuration
  table with defaults), `doc/controllers/*.md` (one section per operation, with a runnable usage example
  and an error table), `doc/models/*.md`, `doc/auth/*.md` (one per auth scheme, when the API has auth),
  `doc/proxy-configuration-builder.md`, plus, when generated, `doc/api-response.md` (the response
  wrapper), `doc/http-request.md`, `doc/file-wrapper.md` and `doc/*logging-configuration-builder.md`.
  `ls doc/` first — the set varies per SDK; the README's own link index at the bottom
  lists exactly what this SDK shipped. **Grep `doc/` first** — it is the fastest way to find an operation
  and a copy-pasteable snippet, then open the `.php` file for the exact signature.

## Placeholder legend

Samples across these skills write `{...}` for a name that differs per API. **They are never literal** —
resolve each one from this SDK's source before you write code. Anything already concrete for this SDK
(Composer package, root namespace, API name) is spelled out for you; if it is still in braces, it is
yours to look up.

| Placeholder | Stands for | Resolve it from |
| --- | --- | --- |
| `{Client}` | the prefix of the client and builder classes, as in `{Client}Builder` | the class file names directly under `src/` |
| `{Postfix}` | the controller class suffix | the class names in the controller directory — `Controller` unless this SDK renamed the postfix, and that one name covers the folder, the classes and the client getters together, so `ls src/` rather than assuming either way |
| `{Group}`, `{group}` | a resource group, as used in the client getter `get{Group}{Postfix}()` | the class names in the controller directory, and the file names under `doc/controllers/` |
| `{Resource}` | a controller class | the same place — `{Group}` plus `{Postfix}` |
| `{operation}` | an operation method on a controller | the controller class body, or the *Parameters* table in `doc/controllers/{group}.md` |
| `{ReturnType}` | what the operation returns | that operation's signature — see **php-calling-endpoints** for which of the two shapes this SDK returns |
| `{Model}` | a generated model class | `src/Models/` |
| `{Field}`, `{optionalField}`, `{ItemsField}` | one property of a model, via `set{Field}(...)` | that model's setters |
| `{jsonName}` | the **wire** name of a property, which often differs from the PHP name | the `@maps` annotation on that property |
| `{EnumClass}`, `{CONSTANT}` | a generated enum class and one of its constants | `src/Models/` |
| `{Variant}`, `{Typed}`, `{Type}`, `{type}` | a union variant, or a typed exception class | `src/Models/` for unions, `src/Exceptions/` for exceptions |
| `{Scheme}`, `{scheme}` | an auth scheme, as it names the credentials class and the builder setter `->{scheme}Credentials(...)` | `src/Authentication/`, and `doc/auth/` |
| `{Param}` | one parameter of an auth scheme, as it names the manager's `get{Param}()` getter | the getters on that scheme's manager in `src/Authentication/` — a parameterless custom scheme has none |
| `{ScopeClass}` | the generated OAuth scope enum | the `## Scopes` table in `doc/auth/*.md` |
| `{slug}`, `{ALIAS}`, `{LOCAL_CONSTANT}`, `{optionalParam}` | a stand-in value in an example | nothing to look up — substitute your own |

Any other `{...}` you meet is a local example; the sentence around it says what belongs there.

## Integration workflow — load the companion skill at each step

**Load the skill named for a step before you write that step's code, even where you have already read
the source.** The generated source is authoritative for the SDK's *surface*; these skills carry the
usage rules a signature cannot show, and each one names the trap that surface hides.

1. **php-client-initialization** — before you construct the client.
2. **php-authentication** — before you set credentials, and when a call returns 401 or 403.
3. **php-calling-endpoints** — before the first operation call.
4. **php-models** — as soon as a request or response field is not a plain string or number.
5. **php-error-handling** — before your first `try`/`catch` around a call.
6. **php-configuration-resilience** — before you ship. Its subject is the client's *defaults*, which apply whether or not you set anything, so this step is not conditional on changing one.
7. **php-testing** — before you stub the SDK.

## Next

- Step 1, construct the client → **php-client-initialization**

Every skill below ends with the step after it, so the footers walk the workflow above in order rather than offering a menu beside it.
