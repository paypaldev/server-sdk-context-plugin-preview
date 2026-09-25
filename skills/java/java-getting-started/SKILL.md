---
name: 'java-getting-started'
description: 'Entry point for integrating the PayPal Server SDK Java SDK (APIMatic-generated, io.apimatic runtime over OkHttp3 and Jackson). Carries the SDK''s identity — Maven coordinates, client class, environments, base exception — the package layout, and where to read the source. Load this first, before any other java-* skill.'
---

# Getting started with an APIMatic-generated Java SDK

This is the **SDK-specific** reference and grounding layer, and the entry into the companion `java-*`
skills. The SDK is built on the shared `io.apimatic` runtime artifacts (`io.apimatic:core`, `io.apimatic:core-interfaces`, and
`io.apimatic:okhttp-client-adapter`), with **OkHttp3** as the transport and **Jackson** for JSON. For the
general patterns that apply to *any* such SDK (client setup, auth, calling endpoints, models, error
handling, retries, testing), see the companion API-agnostic skills: `java-client-initialization`,
`java-authentication`, `java-calling-endpoints`, `java-models`, `java-error-handling`,
`java-configuration-resilience`, `java-testing`.

> **Before writing any integration code, locate the SDK source on disk** (see the *SDK source* section
> below) and read it to confirm every signature, class, enum, and exception type as you go. The source
> is obtained exactly as that section describes. Do **not** vendor or edit it inside your own packages;
> regenerate the SDK instead if something needs to change.

## SDK identity

| | |
| --- | --- |
| API | `PayPal Server SDK` |
| Runtime dependencies | `io.apimatic:core`, `io.apimatic:core-interfaces`, `io.apimatic:okhttp-client-adapter` (version ranges pinned in `pom.xml`); Jackson and OkHttp3 arrive transitively |
| Maven coordinates | `com.paypal.sdk:paypal-server-sdk:2.4.0` — the `<groupId>`/`<artifactId>`/`<version>` in `pom.xml` |
| Client | a single `public final class {Api}Client`, built with `new {Api}Client.Builder()...build()` — there is **no public constructor**. This SDK ships no generated interfaces, so it is declared `implements Configuration` and no client interface is generated |
| Configuration | `Configuration` is a **read-only interface** the client implements (`getEnvironment()`, `getHttpClientConfig()`, `getBaseUri()`), *not* an options object you pass in. Everything is set on the `Builder` |
| Auth | **this API declares at least one security scheme**, so `<root>/authentication/` exists and the client builder carries a credential setter per scheme: a `{Scheme}Model` built with its own nested `Builder` and handed to `.{scheme}Credentials(model)` for most schemes, but an API whose **only** scheme is an OAuth 2 grant drops the suffix (`.clientCredentialsAuth(model)`, `.authorizationCodeAuth(model)`). This is **not** the no-auth case, so read the real setter names off `{Api}Client.java` rather than concluding there is none. See **java-authentication** |
| Environment | an `enum Environment` in the root package — read it for the real member names |
| Servers | an `enum Server` in the root package; `client.getBaseUri(Server)` resolves environment + server to a URL |
| Controllers | accessors on the client, named `get` + the controller class name (e.g. `client.get{Controller}()`) — **the class suffix and the package the classes sit in are both named per SDK**, so read the client's `import` block and its `get...()` accessors for the real names. This SDK ships no generated interfaces, so the accessor's declared type is the `final` implementation class itself and no controller interface is generated |
| Return type | `ApiResponse<T>` — this SDK returns **complete responses**, so every non-paginated operation wraps its value (`getResult()`/`getStatusCode()`/`getHeaders()`, defined in `<root>/http/response/ApiResponse.java`), a body-less operation included (`ApiResponse<Void>`). The `...Async()` twin (generated in asynchronous mode) matches: `CompletableFuture<ApiResponse<T>>`. see **java-calling-endpoints** |
| Base error | `ApiException` in `<root>.exceptions`, a **checked** exception declared on most synchronous operations alongside `IOException`. This build generates at least one typed subclass under `<root>/exceptions/` — per documented error with a modelled body, plus the OAuth provider error on grant builds — and a typed subclass must be caught *before* `ApiException`. See **java-error-handling** |

Both were recorded with this SDK, so they are the coordinates a consumer installs rather than whatever this build happened to produce. Pin them as they stand.

## Package layout

APIMatic Java SDKs live under `src/main/java/<rootPackage>/`:

- `<root>/{Api}Client.java` — the client class, its nested `Builder`, `newBuilder()`, the controller
  accessors, and the private base-URL resolver (`environmentMapper`).
- `<root>/Configuration.java` — the read-only configuration interface the client implements.
- `<root>/Environment.java`, `<root>/Server.java` — the environment and server enums.
- `<root>/ApiHelper.java` — `extends io.apimatic.core.utilities.CoreHelper`; serialization/deserialization
  helpers. `<root>/DateTimeHelper.java` appears only when the API uses a date or date-time type.
- `<root>/<controllerPackage>/` — the base controller plus one `{Controller}` per API group. **The
  package name and the class suffix are both named per SDK** (`controllers`/`Controller` by default), so
  confirm each from the client's `import` block and its `get...()` accessors. This SDK ships no generated interfaces, so it holds one class per group and no interface.
- `<root>/models/` — request/response classes and enums; `<root>/models/containers/` for oneOf/anyOf
  union types. **This is where field names and optionality live, along with enum values.**
- `<root>/exceptions/` — `ApiException` plus a typed subclass per error response with a modelled body (an OAuth grant adds the provider's own).
- `<root>/authentication/` — `{Scheme}Model` (the credentials holder) and `{Scheme}Manager`. The
  credentials *interface* sits in the root package when the API has a single scheme, and in
  `<root>/authentication/` when it has several.
- `<root>/http/client/` — `HttpCallback`, `HttpClientConfiguration` (+ its `Builder`), `HttpContext`,
  `HttpProxyConfiguration`, and the `Readonly*` views.
- `<root>/http/request/` — `HttpRequest`, `HttpBodyRequest`, `HttpMethod`.
  `<root>/http/response/` — `HttpResponse`, `HttpStringResponse`, plus `ApiResponse`, since this build is in complete-response mode.
- `<root>/utilities/` — `FileWrapper`, the wrapper used for file parameters. Always generated, whether
  or not the API has any. There is no `<root>/utilities/pagination/`: this API declares no paginated operation.
- `<root>/logging/configuration/` — **present**: this SDK ships logging, so the client `Builder` also carries `loggingConfig(...)`. Call it with **no arguments** for the default stdout logger; the lambda overload needs `org.slf4j.event.Level` for `.level(...)`, which is the import a reader reaching for logging here otherwise has to go and find. See **java-configuration-resilience**.
- `src/test/java/<root>/testing/HttpCallbackCatcher.java` — present **only** when the SDK was generated
  with tests. See **java-testing**.

## Install

This SDK is published out of `https://github.com/paypal/PayPal-Java-Server-SDK`, branch `main` — take that branch explicitly rather than the repository default, which is not necessarily where this SDK is released from. Which registry that pipeline pushes to is a property of the pipeline rather than of the SDK. Try the install command below as-is first: if it resolves, the package is on the public registry and there is nothing further to configure. Only if it 404s do you need the feed — take it from the repository's publish workflow, or from whoever owns the pipeline, and configure that registry before retrying.

The SDK is a Maven source project. Add it as a normal dependency:

```xml
<dependency>
    <groupId>com.paypal.sdk</groupId>
    <artifactId>paypal-server-sdk</artifactId>
    <version>2.4.0</version>
</dependency>
```

Gradle consumers use the same coordinates:

```groovy
implementation 'com.paypal.sdk:paypal-server-sdk:2.4.0'
```

Then make the artifact resolvable the way this SDK was distributed:

```bash
# Make the artifact resolvable, then build normally:
mvn dependency:get -Dartifact=com.paypal.sdk:paypal-server-sdk:2.4.0
```

A rebuilt SDK carries a new version, and a stale `<version>` in the consuming `pom.xml` keeps resolving
happily, so confirm what actually resolved:

```bash
mvn dependency:tree -Dincludes=com.paypal.sdk:paypal-server-sdk
```

```java
import {rootPackage}.{Api}Client;
import {rootPackage}.Environment;
import {rootPackage}.exceptions.ApiException;
import {rootPackage}.{controllerPackage}.{Group}{Postfix};
import {rootPackage}.models.{Model};
```

Imports come from the real root package — take it from the `<Automatic-Module-Name>` in `pom.xml` or the
`package` line at the top of any generated file. Copy `{controllerPackage}` and `{Postfix}` straight from
the `import` lines at the top of `{Api}Client.java`, which imports every controller by its actual package
and class name.

## SDK source — read it in place

You will constantly need to confirm real builder methods, controller accessors, operation signatures,
model classes, enum values, and exception types, and the **only reliable way** is to read the SDK source.

Clone it and read the clone — it is the source this pack documents:

```bash
git clone --filter=blob:none --branch main https://github.com/paypal/PayPal-Java-Server-SDK
```

**The clone is a branch; your install is pinned.** Check out the tag matching `2.4.0`, the version installed above before you read anything from it — a branch keeps moving after a release is cut, so the default checkout can be a different SDK than the one you compile against, and nothing in the tree will tell you. `git ls-remote --tags` lists what the repository offers; if no tag matches, treat every signature you read as unconfirmed rather than assuming it carried over. Clone it outside your project directory and treat it as read-only. It is a reference, not a dependency: what you build against is the package installed above, never this checkout.

An existing copy, if you already have one, is in whichever of these applies:

- the **unpacked SDK directory** you were given (the one containing `pom.xml` and `src/main/java/`); or
- the sources jar in your local Maven repository under
  `~/.m2/repository/<groupId as directories>/paypal-server-sdk/2.4.0/`.

Treat it as a read-only reference and grep it locally.

Layout — grep here first:

- `pom.xml` — the GAV coordinates, the `io.apimatic` dependency versions, and the Java compile target.
- `<root>/{Api}Client.java` — the `Builder` methods (including the auth credential setters), the
  controller accessors, and `environmentMapper` (the real base URLs per environment).
- `<root>/Environment.java`, `<root>/Server.java` — the real enum member names.
- `<root>/<controllerPackage>/*.java` — operation methods, their parameters, and their `throws` clauses.
  The package name is named per SDK (`controllers` by default); get the real one from the `import` lines
  at the top of `{Api}Client.java`.
- `<root>/models/` — model classes, their `@JsonGetter`/`@JsonSetter` wire names, and `containers/`
  unions; **this is where field names live**.
- `<root>/exceptions/` — `ApiException` and the typed subclasses.
- `README.md` and `doc/` — a generated, human-readable index: `doc/client.md`,
  `doc/controllers/*.md` (that folder can differ from the source package), `doc/models/*.md`, `doc/auth/*.md`,
  `doc/api-exception.md`, `doc/http-client-configuration-builder.md`, `doc/http-callback-interface.md`,
  `doc/http-context.md`,
  `doc/configuration-interface.md`. In the repository, it is the fastest way to find an operation,
  its parameters, and a copy-pasteable usage snippet, then open the `.java` file for the exact signature.
  **It is in no jar**, so resolving the dependency gives you no `doc/` at all — with only the sources
  jar, the controller classes below are the equivalent starting point.

## Placeholder legend

Samples across these skills write `{...}` for a name that differs per API. **They are never literal** —
resolve each one from this SDK's source before you write code. Anything already concrete for this SDK
(artifact coordinates, API name, install snippet) is spelled out for you; if it is still in braces, it is
yours to look up.

| Placeholder | Stands for | Resolve it from |
| --- | --- | --- |
| `{rootPackage}` | the SDK's root Java package | the `package` line at the top of any generated `.java` file |
| `{Api}` | the prefix of the client class, as in `{Api}Client` | the client class file name beside the root package |
| `{controllerPackage}` | the sub-package holding controllers | the folder beside the client class — `controllers` on many SDKs and something else on others, so `ls` it rather than assuming |
| `{Controller}`, `{Group}`, `{group}` | a controller class and the resource group it covers (`{group}` is the lower-case form used in doc paths) | the class names in `{controllerPackage}`, and the file names in `doc/controllers/`. The class postfix is named per SDK |
| `{operation}`, `{Operation}` | an operation method, and the same name capitalised for types built from it | the controller class body, or `doc/controllers/{Group}.md` |
| `{params}` | the operation's full parameter list | that operation's signature |
| `{ReturnType}` | what the operation returns | the same signature — always an `ApiResponse<...>` in this build; see **java-calling-endpoints** |
| `{Model}`, `{BodyModel}`, `{Parent}` | a generated model class, a request body, a polymorphic base | `{rootPackage}.models` |
| `{Field}`, `{field}` | one property of a model | that model's Builder and getters |
| `{EnumType}`, `{MEMBER}` | a generated enum and one of its members | `{rootPackage}.models`; `{MEMBER}` also covers `Environment` members in `Environment.java` |
| `{Union}`, `{Variant}`, `{VariantAType}`, `{VariantBType}`, `{variantA}`, `{variantB}` | a oneOf/anyOf container and its variants | `{rootPackage}.models.containers` |
| `{PageType}`, `{Item}` | a paginated wrapper and its element type | the paginated operation's signature |
| `{Scheme}`, `{scheme}`, `{SchemeName}`, `{X}` | an auth scheme, as it names credential classes and client getters | `{rootPackage}.authentication`, and `doc/auth/` |
| `{OAuthScopeEnum}`, `{SCOPE}` | the generated OAuth scope enum and a member | the `## Scopes` table in `doc/auth/*.md` |
| `{CONSTANT}`, `{placeholder}` | a stand-in value in an example | nothing to look up — substitute your own |

Any other `{...}` you meet is a local example; the sentence around it says what belongs there.

## Integration workflow — load the companion skill at each step

**Load the skill named for a step before you write that step's code, even where you have already read
the source.** The generated source is authoritative for the SDK's *surface*; these skills carry the
usage rules a signature cannot show, and each one names the trap that surface hides.

1. **java-client-initialization** — before you construct the client.
2. **java-authentication** — before you set credentials, and when a call returns 401 or 403.
3. **java-calling-endpoints** — before the first operation call.
4. **java-models** — as soon as a request or response field is not a plain string or number.
5. **java-error-handling** — before your first `try`/`catch` around a call.
6. **java-configuration-resilience** — before you ship. Its subject is the client's *defaults*, which apply whether or not you set anything, so this step is not conditional on changing one.
7. **java-testing** — before you stub the SDK.

## Next

- Step 1, construct the client → **java-client-initialization**

Every skill below ends with the step after it, so the footers walk the workflow above in order rather than offering a menu beside it.
