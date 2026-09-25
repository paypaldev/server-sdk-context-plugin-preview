---
name: 'csharp-getting-started'
description: 'Entry point for integrating the PayPal Server SDK C# SDK (APIMatic-generated, built on APIMatic.Core with Newtonsoft.Json). Carries the SDK''s identity — package id, client class, environments, base exception — this SDK''s own map of schemes, environments and controllers, the package layout, and where to read the source. Load this first, before any other csharp-* skill.'
---

# Getting started with an APIMatic-generated C# SDK

This is the **SDK-specific** reference and grounding layer, and the entry into the companion `csharp-*`
skills. The SDK is a thin shim over the **`APIMatic.Core`** runtime package, with
**Newtonsoft.Json** for serialization. For the general
patterns that apply to *any* such SDK (client setup, auth, calling endpoints, models, error handling,
retries, testing), see the companion API-agnostic skills: `csharp-client-initialization`,
`csharp-authentication`, `csharp-calling-endpoints`, `csharp-models`, `csharp-error-handling`,
`csharp-configuration-resilience`, `csharp-testing`.

> **Before writing any integration code, locate the SDK source on disk** (see the *SDK source* section
> below) and read it to confirm every signature, class, enum, and exception type as you go. The source
> is obtained exactly as that section describes. Do **not** copy or edit it inside your own source tree;
> regenerate the SDK instead if something needs to change.

## SDK identity

| | |
| --- | --- |
| API | `PayPal Server SDK` |
| Runtime dependencies | `APIMatic.Core`, `Microsoft.CSharp`, `Microsoft.Extensions.Configuration.Binder`; a build adds more — **read the `PackageReference` rows in the `.csproj`**. Newtonsoft.Json arrives **transitively** |
| Package id | `PayPalServerSDK` (version `2.4.0`) — the `<AssemblyName>`, and the only string `dotnet add package` accepts |
| Install | `dotnet add package PayPalServerSDK` |
| Root namespace | `PaypalServerSdk.Standard` — **also the project name, folder and `.csproj` file name**, and what every `using` references |
| Target framework | `netstandard2.0` |
| Client | one `public sealed class PaypalServerSdkClient`, built with `new PaypalServerSdkClient.Builder()…Build()` — the constructor is **private** |
| Configuration | `IConfiguration` in `PaypalServerSdk.Standard` is a **read-only view** the client implements, not an options object you pass in; `PaypalServerSdkClient.FromConfiguration(configuration.GetSection("PaypalServerSdk"))` binds one |
| Auth | see the **Authentication** table below — the setter names are resolved for this SDK, not a pattern to derive. **If that table says this SDK declares no authentication scheme, it needs none** — there is no `Authentication/` folder and no credential setter on the `Builder`, so do not invent one. Details in **csharp-authentication** |
| Environment | enums `Environment` and `Server` in `PaypalServerSdk.Standard`, resolved by `client.GetBaseUri(Server)`. `Environment` **collides with `System.Environment`**, and there is **no base-URL setter** |
| Controllers | get-only properties on the client, one per API group, in `PaypalServerSdk.Standard.Controllers`; their constructors are `internal`. **Take the property names off the client** |
| Return type | `ApiResponse<T>` (`StatusCode`, `Headers`, `Data`) — this SDK wraps its responses; `void`/`Task` for empty responses. **An operation whose response the spec never modelled is declared `dynamic` or `object` instead — and that value is the raw undeserialized body stream, not an object.** Which of the two varies per build. Census it before planning any call: `grep -c 'Task<dynamic>\|Task<object>' Controllers/*.cs` — on some SDKs it is most of the surface |
| Base error | `ApiException` in `PaypalServerSdk.Standard.Exceptions`, extending `CoreApiException<HttpRequest, HttpResponse, HttpContext>` — see **csharp-error-handling** |

Both were recorded with this SDK, so they are the coordinates a consumer installs rather than whatever this build happened to produce. Pin them as they stand.

## This SDK's map

Resolved at generation time — these are real names, not placeholders.

### Authentication

| Scheme | Builder setter | Credential model | Interface | Client getter | Config bind type |
| --- | --- | --- | --- | --- | --- |
| `ClientCredentialsAuth` | `.ClientCredentialsAuth(...)` | `ClientCredentialsAuthModel.Builder(oAuthClientId, oAuthClientSecret)` | `IClientCredentialsAuth` | `client.ClientCredentialsAuth` | `ClientCredentialsAuthModelOptions` |

> The **Config bind type** column is the class `FromConfiguration` binds into. Its casing can differ from
> the credential model's — one SDK emits `VZM2mTokenModel` but `VZM2MTokenModelOptions` — so take it from
> this table rather than appending `Options` to the model name.
>
> The setter is **not** always the scheme name plus `Credentials`: the suffix is dropped where the SDK
> has exactly one scheme *and* it is an OAuth 2 grant, and where a scheme is already named for a grant
> the word **doubles** (`.Oauth2ClientCredentialsCredentials(...)`). Neither is a typo in the table
> above — copy from it rather than reasoning about it; the model getter is always `client.{Scheme}Model`.

### Environments

| Environment member | Servers it defines |
| --- | --- |
| `Environment.Production` | `Server.Default` |
| `Environment.Sandbox` | `Server.Default` |

The `Environment` member selects the environment; each operation picks its own `Server` alias internally,
so `client.GetBaseUri(Server.X)` reports a URL but does not route a call. **"Servers it defines" is not
"servers you can reach"** — a `Server` member can be declared by an environment and pinned by no operation
at all, in which case its URL is reachable only through `GetBaseUri`. `grep -rn '\.Server(' Controllers/`
shows which ones any operation actually selects; everything else in the table is diagnostics.

### Configuration variables

**This SDK declares no configuration variables.** The `Builder` exposes only credential setters, `Environment`, `HttpClientConfig`, `HttpCallback`, `Build` and `FromConfiguration` — there is nothing else to set, and no `.{ConfigVar}(...)` method to look for.

### Controllers

`client.OrdersController`, `client.PaymentsController`, `client.SubscriptionsController`, `client.TransactionSearchController`, `client.VaultController`

These are the **business** controllers, one per endpoint group. An SDK with an OAuth scheme also exposes an
authorization controller — named for the scheme and carrying the same postfix as the rest, so
`OAuthAuthorization` plus whatever suffix the list above uses — which is **not** in this list: it is the
token endpoint the auth manager drives for you, not an API to call yourself. Note that `doc/client.md`'s
own Controllers table *does* list it.

Operation names, parameters and model fields are still per-operation — read the controller `.cs` file, as
the sections below direct.

## Package layout

The SDK is a .NET solution: a `.sln` at the archive root beside `README.md`, `LICENSE` and `doc/`. The
**class library you reference is `PaypalServerSdk.Standard/`**; a generated NUnit test project sits beside it
when the SDK was built with tests. Everything below is inside the class-library folder, and `{...}`
names such as `{Scheme}`, `{Controller}`, `{Variant}` and `{Field}` are placeholders you replace with
the concrete identifier from the source.

- `PaypalServerSdk.Standard.csproj` — target framework, `<AssemblyName>` (the package id), `PackageReference`s.
- `PaypalServerSdkClient.cs` — the client and its nested `Builder`, with `IConfiguration.cs`, `Environment.cs`
  and `Server.cs` beside it.
- `Controllers/` — one controller class per API group, plus a base class. **The folder name and
  the class suffix vary between SDKs**, so take the real names off the
  client's properties.
- `Models/` — model classes and enums, with `Models/Containers/` for oneOf/anyOf unions. **This is where
  field names, optionality, `[JsonProperty]` names and `[EnumMember]` values live.**
- `Exceptions/` — `ApiException`, plus typed subclasses where the spec models an error body. **Not one per
  status**: several documented statuses routinely share a single subclass, and some SDKs emit none at all,
  so do not plan a catch arm per status. The `ErrorCase` registrations are the status-to-class map — see
  **csharp-error-handling**.
- `Authentication/` — per scheme, `{Scheme}Manager.cs` (which also declares `{Scheme}Model`, its nested
  `Builder` and `{Scheme}ModelOptions` — there is **no** `{Scheme}Model.cs`) and `I{Scheme}Credentials.cs`.
- `Http/` — `Client/` holds `HttpClientConfiguration`, `HttpCallback`, `HttpContext`, `FileStreamInfo`
  and `Proxy/`; `Request/` and `Response/` hold `HttpRequest` and `HttpResponse`,
  plus `ApiResponse.cs`, emitted because this SDK wraps responses.
- `Utilities/` — `ApiHelper` and `CompatibilityFactory` always; the date converters, `Pagination/` and
  `AdditionalPropertiesExtensions` only when the API needs them.

  > **`ApiHelper` is an empty subclass — do not read its file to decide what it offers.** It is
  > generated as `public class ApiHelper : CoreHelper { }`, a dozen lines with no members of its own.
  > Every method you call on it is **inherited from `CoreHelper` in the `APIMatic.Core` runtime
  > package**, and C# resolves an inherited public static member through the derived type name, so
  > `ApiHelper.{Member}(...)` compiles even though `ApiHelper.cs` mentions nothing. A bare file is not
  > evidence the member is missing: resolve the base type and look there, or let the compiler answer.
- `Logging/LogBuilder.cs` — present, because this SDK has built-in logging.

- `Examples/` — **only in some builds**: per-operation snippets in namespace `TestConsoleProject`,
  compiled into the class library. `ls` for it. When present it is useful grounding — each snippet is one
  real call with real member names — and it is also where a generation defect surfaces first: if
  `dotnet build` fails on files you did not write, look here before anything else.

> **Model names can shadow BCL types.** A spec entity called `Task`, `Environment`, `Console` or
> `Convert` emits `Models/{Name}.cs`, so a blanket `using PaypalServerSdk.Standard.Models;` then makes the bare
> name ambiguous — a generated `Task` model breaks every file that also writes `async Task` (`CS0104`). `ls Models/`
> for BCL names before importing the namespace. **Do not fix it by aliasing the BCL type**
> (`using Task = System.Threading.Tasks.Task;`) — that compiles, but it aliases the *SDK* type out of
> reach, so the model becomes nameable only by its full namespace, which is the opposite of what you
> wanted. Instead do not import `PaypalServerSdk.Standard.Models` at all: alias the namespace
> (`using Models = PaypalServerSdk.Standard.Models;`) and write `Models.{Model}`, leaving bare `Task` as the BCL
> type. That is the same form **csharp-models** and **csharp-calling-endpoints** use.

**Absence is meaningful for the folders above** — `ls` rather than assume a class exists.

## Install

This SDK is published out of `https://github.com/paypal/PayPal-Dotnet-Server-SDK`, branch `main` — take that branch explicitly rather than the repository default, which is not necessarily where this SDK is released from. Which registry that pipeline pushes to is a property of the pipeline rather than of the SDK. Try the install command below as-is first: if it resolves, the package is on the public registry and there is nothing further to configure. Only if it 404s do you need the feed — take it from the repository's publish workflow, or from whoever owns the pipeline, and configure that registry before retrying.

```bash
dotnet add package PayPalServerSDK
```

If the SDK was published, that is a `dotnet add package` line resolving `PayPalServerSDK` from your feeds.
If it was not, the SDK is a **source project, not a registry package** — reference the class library's
`.csproj` instead, from under `vendor/` in the consuming project, committed but never edited:

```xml
<ItemGroup>
  <ProjectReference Include="vendor/PaypalServerSdk.Standard/PaypalServerSdk.Standard.csproj" />
</ItemGroup>

<PropertyGroup>
  <DefaultItemExcludes>$(DefaultItemExcludes);vendor/**</DefaultItemExcludes>
</PropertyGroup>
```

Reference the **class library, not the `.sln` and not any test project**. The `<DefaultItemExcludes>`
line is not optional — without it the consumer sweeps the vendored `*.cs` into its own compile and the
build fails with `CS0579: Duplicate 'AssemblyTitleAttribute' attribute`. Keep the
`$(DefaultItemExcludes);` prefix, or the defaults are replaced rather than extended. Then import by
namespace:

```csharp
using PaypalServerSdk.Standard;
using PaypalServerSdk.Standard.Models;
using Environment = PaypalServerSdk.Standard.Environment;   // else CS0104 wherever `using System;` is present
```

When the SDK **is** published, the project-file equivalent of that install line is a pinned package
reference. Pin it: regenerating the SDK publishes a new version, and a floating reference moves the
surface without saying so.

```xml
<ItemGroup>
  <PackageReference Include="PayPalServerSDK" Version="2.4.0" />
</ItemGroup>
```

Confirm what restored rather than assuming:

```bash
dotnet list package
```

### `PayPalServerSDK` and `PaypalServerSdk.Standard` are not interchangeable

`PayPalServerSDK` belongs in `dotnet add package` and `<PackageReference Include="…">` and **nowhere
else**. `using PayPalServerSDK;` does not compile.

## SDK source — read it in place

You will constantly need to confirm real builder methods, controller property names, operation signatures,
model classes, enum members and exception types, and the **only reliable way** is to read the SDK source.

Clone it and read the clone — it is the source this pack documents:

```bash
git clone --filter=blob:none --branch main https://github.com/paypal/PayPal-Dotnet-Server-SDK
```

**The clone is a branch; your install is pinned.** Check out the tag matching `2.4.0`, the version installed above before you read anything from it — a branch keeps moving after a release is cut, so the default checkout can be a different SDK than the one you compile against, and nothing in the tree will tell you. `git ls-remote --tags` lists what the repository offers; if no tag matches, treat every signature you read as unconfirmed rather than assuming it carried over. Clone it outside your project directory and treat it as read-only. It is a reference, not a dependency: what you build against is the package installed above, never this checkout.

An existing copy, if you already have one, is the **unpacked SDK directory** you were given (the one
containing the `.sln`, `README.md`, `doc/` and the `PaypalServerSdk.Standard/` project folder), or the copy
inside your repository if it was wired in with a `<ProjectReference>`.

> **A NuGet install is not that copy.** A `PackageReference` restores a package containing the
> compiled assembly and an XML documentation file — **no `.cs` files and no `doc/` directory.** The
> layout below describes the SDK's *source tree*, which is a different artifact from the restored
> package; the paragraph above this one says where that tree comes from for this SDK.
>
> Until you have it, the only reference that ships with the package is `PayPalServerSDK.xml`, beside the
> assembly in the package folder. That is the IntelliSense file: greppable for member names and summary
> text, carrying no usage examples — so it answers "does this member exist, and what is it called" and
> nothing else.

Treat the source tree as a read-only reference and grep it locally.

Layout — grep here first (paths are inside the `PaypalServerSdk.Standard/` project folder, except the last):

- `PaypalServerSdkClient.cs` — the authoritative list of nested-`Builder` methods, their field-initialiser
  defaults, and the environment-to-base-URL map. **A client-level setter that is not on that `Builder`
  does not exist** — transport settings live on `HttpClientConfiguration.Builder` instead.
- `Environment.cs`, `Server.cs` — the real member names; they come from the spec's environment and server
  lists, so do not assume a particular one exists.
- `Controllers/*.cs` — operation signatures, the sync/async pairs, and the `ErrorCase`
  registrations that say which exception class each status code maps to.
- `Models/` — `[JsonProperty]` wire names, required-vs-optional, enum members and `Containers/` unions;
  **this is where field names live**. `Exceptions/` — `ApiException` and the typed subclasses.
- `Http/Client/HttpClientConfiguration.cs` — the timeout / retry / proxy / custom-`HttpClient` surface.
- `README.md` and `doc/` at the archive root — a generated, human-readable index: `doc/client.md`,
  `doc/controllers/*.md`, `doc/models/*.md`, `doc/auth/*.md`. **Present only in the source tree**, not
  in the installed package. When you have it, grep `doc/` first — it is the fastest way to find an
  operation, its parameters, and a copy-pasteable usage snippet, then open the `.cs` file for the exact
  signature. When you do not, `Controllers/*.cs` is the equivalent starting point, and the
  operation's own signature replaces the snippet.

> **The generated docs are a finding aid, not a contract — the `.cs` file wins.** `doc/` can advertise
> methods that do not exist and signatures that differ from the code (the companion skills name the
> specific cases). Use `doc/` to locate things; confirm every name and signature against the generated
> `.cs` before you write it.

## Placeholder legend

Samples across these skills write `{...}` for a name that differs per API. **They are never literal** —
resolve each one from this SDK's source before you write code. Anything already concrete for this SDK
(package id, root namespace, client class, API name) is spelled out for you; if it is still in braces, it
is yours to look up.

| Placeholder | Stands for | Resolve it from |
| --- | --- | --- |
| `{Controller}` | a controller, read off the client as a property (`client.{Controller}`) | the properties on the client class, and the class names in the SDK's `Controllers/` folder — the folder and the class postfix vary between SDKs, so read both rather than assuming |
| `{group}` | the resource group a controller covers | the file names under `doc/controllers/` |
| `{Operation}`, `{operation}` | an operation, generated as a blocking `{Operation}` alongside an async form | the controller class body, or `doc/controllers/{group}.md` |
| `{pathParam}`, `{OptionalParam}`, `{filter}`, `{since}`, `{file}` | one parameter of the operation you are calling | that operation's signature |
| `{ReturnType}` | what the operation returns | the same signature — see **csharp-calling-endpoints** for how the complete-response setting decides the shape |
| `{Model}`, `{model}`, `{RequestModel}`, `{Parent}` | a generated model class, or its doc page `doc/models/{model}.md` | the SDK's `Models/` folder |
| `{Field}`, `{field}`, `{RequiredField}`, `{OptionalField}`, `{requiredField}`, `{optionalField}`, `{SomeField}`, `{OtherField}`, `{Property}`, `{RequiredProp}`, `{OptionalProp}` | one property of a model, and the constructor argument that sets it | that model's constructor and properties |
| `{wireName}` | the **wire** name of a property, which often differs from the C# name | the `[JsonProperty("...")]` attribute on that property |
| `{EnumType}`, `{Member}`, `{MEMBER}` | a generated enum and one of its members | the SDK's `Models/`; `{MEMBER}` also covers `Environment` members |
| `{Union}`, `{union}`, `{Variant}`, `{VariantA}`, `{VariantB}`, `{variantA}`, `{variantB}`, `{T}` | a oneOf/anyOf container, its variants, and the type you unwrap to | the SDK's `Models/Containers/`, and `doc/models/containers/{union}.md` |
| `{Item}`, `{PagedResponse}`, `{Page}` | a paginated wrapper and its element type — only meaningful on an SDK that paginates | the paginated operation's signature |
| `{Error}`, `{Typed}` | a typed exception class | the SDK's `Exceptions/` folder |
| `{Scheme}` | an auth scheme, as it names `{Scheme}Manager` and its credential interfaces | the SDK's `Authentication/` folder, and `doc/auth/` |
| `{ScopesEnum}`, `{Scope}` | the generated OAuth scope enum and a member | the `## Scopes` table in `doc/auth/*.md` |
| `{ConfigVar}`, `{SERVER}`, `{Default}`, `{Strategy}`, `{xSomeHeader}`, `{discriminatorField}`, `{ChildA}`, `{ChildB}`, `{childAValue}`, `{childBValue}`, `{params}`, `{placeholder}` | a stand-in value or a locally-explained name in an example | the sentence around it, or substitute your own |

Any other `{...}` you meet is a local example; the sentence around it says what belongs there.

## Integration workflow — load the companion skill at each step

**Load the skill named for a step before you write that step's code, even where you have already read
the source.** The generated source is authoritative for the SDK's *surface*; these skills carry the
usage rules a signature cannot show, and each one names the trap that surface hides.

1. **csharp-client-initialization** — before you construct the client.
2. **csharp-authentication** — before you set credentials, and when a call returns 401 or 403.
3. **csharp-calling-endpoints** — before the first operation call.
4. **csharp-models** — as soon as a request or response field is not a plain string or number.
5. **csharp-error-handling** — before your first `try`/`catch` around a call.
6. **csharp-configuration-resilience** — before you ship. Its subject is the client's *defaults*, which apply whether or not you set anything, so this step is not conditional on changing one.
7. **csharp-testing** — before you stub the SDK.

## Next

- Step 1, construct the client → **csharp-client-initialization**

Every skill below ends with the step after it, so the footers walk the workflow above in order rather than offering a menu beside it.
