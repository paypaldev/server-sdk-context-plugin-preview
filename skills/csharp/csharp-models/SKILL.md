---
name: 'csharp-models'
description: 'Work with models from the PayPal Server SDK C# SDK. Load before building a request payload or mapping a response onto your own types. The class won''t tell you which fields serialize on a model you never touched, how optional-and-nullable state is tracked, how a oneOf/anyOf container matches, or where unknown fields land.'
---

# Working with models in an APIMatic C# SDK

Data models are plain classes under `Models/`, namespace `PaypalServerSdk.Standard.Models`. **If** the API
declares a `oneOf`/`anyOf`, containers sit beside them in `Models/Containers/` — `ls` that folder
first, because many APIs declare none and it is then absent. Plain scalars need nothing from this skill — see
**csharp-calling-endpoints**. The emitted shapes are catalogued in [reference.md](reference.md); this
skill is the rules for using them.

## Anatomy of a generated model

A model emits a parameterless constructor and an all-fields one, and `doc/models/*.md` builds its examples
with an object initializer. The all-fields constructor puts the **required parameters first, then the
optional ones**, each group in spec order — so its parameter order is not the spec's field order, and you
cannot read one off the other. **Nor is the C# required/optional split reliable**: a required-but-nullable
field is emitted as a C# optional argument defaulting to `null`, so `new {Model}()` can compile while
leaving required fields unset and serializing them as `null`. Take required-vs-optional from the **Tags**
column of `doc/models/{model}.md`, and prefer the object initializer so every omission is explicit:

> **`Models.{Model}` does not compile in your code.** The skills and the generated source write model
> types that way because that source sits *inside* `PaypalServerSdk.Standard`, where `Models` is a nested
> namespace. A `using` directive does not import nested namespaces, so from a consumer project either
> import the models namespace directly, or alias it —
> `using Models = PaypalServerSdk.Standard.Models;` — which is what you want when a model name also shadows a
> BCL type.

```csharp
using PaypalServerSdk.Standard.Models;
var m = new {Model}
{
    {RequiredField} = value,      // nothing checks that you set it
    {OptionalField} = other,
};
```

Prefer the initializer to the all-fields constructor: that constructor's argument order shifts the
moment the API is regenerated with a new required field. Three things to read off the class file
rather than guess:

| What you read | What it tells you |
| --- | --- |
| `[JsonProperty("{wireName}")]` | the **wire name** — **never derive the C# name from it.** It is usually the PascalCase rendering (`trackingNumber` → `TrackingNumber`), but a rendering that lands on a C# keyword, a reserved word, or the declaring type's own name is prefixed with `M`: wire `value` → `MValue`, `string` → `MString`, `Event` → `MEvent`. Exception classes rename on a different collision list — see **csharp-error-handling** |
| `NullValueHandling = NullValueHandling.Ignore` on that attribute | plain-optional — leave the property `null` and the key is omitted |
| a `ShouldSerialize{Field}()` method on the class | the property is optional **and** nullable — see the next section |
| `NullValueHandling.Include`, with **no** `ShouldSerialize`/`Unset` pair — **or, more commonly, no `NullValueHandling` argument at all**, which is how a plain required field is emitted | **a required field** — the key is always emitted, `null` included, and there is no way to drop it. This is the shape most often misread: the C# argument is a *nullable optional parameter defaulting to `null`*, so the compiler lets you omit it and the SDK then ships `"field":null` for something the spec requires. **Neither `Include` nor a bare `[JsonProperty]` is an *optional* marker** — but do not go looking for `Include` specifically: plenty of SDKs never emit it, and there required fields carry the bare attribute alone. Confirm against the **Tags** column of `doc/models/{model}.md` |

**Two model shapes come out of the same generator, and only the class file says which you have.**
Everything above is the default one. Generated with immutable models a class carries `{ get; }`-only
properties and no parameterless constructor, so an object initializer will not compile — construct
through the public nested `Builder` instead. Both shapes are in [reference.md](reference.md).

## Optional vs optional-and-nullable — the `Unset{Field}()` rule

**Check the surface exists before reading on:** `grep -rl "ShouldSerialize" Models/` — or just open one model. No hits means this SDK emits **no** `ShouldSerialize`/`Unset` pair on any
model, no property carries a default, and nothing below applies: an optional field you never set is
simply absent from the payload, and there is nothing to un-set. That is a common shape.

Two different things get called "optional", and C# spells them the same way.

- **Optional** — a plain auto-property whose attribute carries
  `NullValueHandling = NullValueHandling.Ignore`. Leave it `null` and the key never reaches the wire.
- **Optional and nullable** — a hand-written getter/setter over a private backing field, plus
  `ShouldSerialize{Field}()` and `Unset{Field}()` (never on an exception class — see
  **csharp-error-handling**). **The setter turns the serialize flag on unconditionally**, so
  assigning `null` is not the same as leaving the property alone:

```csharp
m.{Field} = null;               // sends the key with a JSON null
m.Unset{Field}();               // drops the key from the payload entirely
m.ShouldSerialize{Field}();     // is this key currently on the wire?
```

The all-fields constructor assigns an optional-and-nullable argument **only when it is non-null**, so
`new {Model}(requiredA, optionalC: null)` leaves the flag off and omits the key; a plain-optional
argument is assigned unconditionally.

> **The serialize flag is not always off to begin with.** A field with a **default value** has its flag
> initialised to `true`, so the key ships with that default on a model you never touched — `new {Model}{
> Required = "x" }` can serialize `"{field}":"{fieldDefault}"`. `Unset{Field}()` is then the *only* way
> to take it back off the wire.
> Two consequences worth checking before you trust a payload:
>
> - `ShouldSerialize{Field}()` on a fresh object can be `true`. Read it rather than assuming `false`.
> - Some SDKs also carry a second map (`hasPropertySetterCalledFor`) driving a getter that returns the
>   field's default instead of `null` when nothing has assigned the property — so `if (m.{Field} == null)`
>   is not a reliable "was it set?" test either.
>
> **Do not treat "no `shouldSerialize` entry is `true`" as "nothing ships by default"** — a
> required non-nullable enum or scalar also ships its zero value on an untouched model, by the separate rule
> below. To see what actually goes out, dump `ApiHelper.JsonSerialize(model)`
> (`PaypalServerSdk.Standard.Utilities`) for a model you have barely populated.
>
> **That dump throws on a model carrying `[JsonRequired]`** — `Cannot write a null value for property
> '{field}'. Property requires a value.` — so populate every `[JsonRequired]` member first, or serialize a
> model that has none. `grep -rl JsonRequired Models/` tells you which are affected; it is usually the union
> variants. Grepping for `shouldSerialize` entries initialised `true`
> (`grep -rn '{ "[a-z_0-9]*", true }' Models/`) covers only the armed-flag route, not the others.

Required scalars and enums are non-nullable value types; optional ones become `T?`. **Nullable
reference types are not enabled**, so a `string` carries no `?` either way and a required one can
still be `null` at runtime. The **Tags** column of `doc/models/{model}.md` marks the required fields.

## Enums

Generated enums are **real C# `enum` value types**, not classes of constants. The wire value lives in
`[EnumMember]` and the converter sits on the type, so you pass and receive members and the SDK handles
the string form.

- **The member name is not the wire value.** `{EnumType}.{Member}` serializes as its `[EnumMember]`
  string while `.ToString()` yields the C# name. For the wire string use
  `ApiHelper.JsonSerialize(value).Trim('"')` (`ApiHelper` is in `PaypalServerSdk.Standard.Utilities`) — what the controllers do for enum query and form
  parameters.
- **A required enum property has no unset state.** `default({EnumType})` is `0`, the first declared
  member of a string-backed enum, so an enum you forget to assign silently serializes as that member
  instead of failing. Optional enum properties are `{EnumType}?` and do distinguish unset.
- An enum whose spec allows values outside the declared set gains an extra `_Unknown` member and a
  wrapped converter; **check for it before writing an exhaustive `switch`**. Integer-backed enums
  carry no `[EnumMember]` at all, and their `default` may match no declared member.
- Read `Models/{EnumType}.cs` for the real member names; never derive them from the wire strings. The
  three declaration shapes are in [reference.md](reference.md).

## oneOf / anyOf union types

**This section applies only if `Models/Containers/` exists** — check before reading on; an API with no
`oneOf`/`anyOf` has no containers and no `PaypalServerSdk.Standard.Models.Containers` namespace, so the
`using` below would not compile.

A field that can be one of several types becomes an **abstract container class** under
`Models/Containers/` — no public constructor, no implicit conversion. The `UnionTypeConverter`
attribute on the class names every case and ends with `true` for `oneOf`, `false` for `anyOf`.

### Construct — a static factory, never a constructor

```csharp
using PaypalServerSdk.Standard.Models;              // the variant models
using PaypalServerSdk.Standard.Models.Containers;   // only exists when the API declares a oneOf/anyOf
{Union} u = {Union}.From{VariantA}(new {VariantA} { /* … */ });
```

**The factories validate nothing.** A `oneOf` that matches two variants, or none, is rejected while a
*response* is deserialized, not while you build the request — see **csharp-error-handling**.

### Read — `Match`, or `MatchSome` when you mean it

```csharp
string label = u.Match(a => "A " + a.{SomeField}, b => "B " + b.{OtherField});
string maybe = u.MatchSome(a => a.{SomeField});   // unhandled cases yield default(T)
```

`Match` takes one callback per case. `MatchSome` takes only the callbacks you want and **returns
`default(T)` — `null` or `0` — for a case you did not handle**, so an unhandled variant is silent;
prefer `Match`. Every instance is really a `private sealed {Variant}Case` nested in the container — a
type you cannot name — so there is nothing to `switch` on and no `Is{Variant}`/`Get{Variant}`
accessors. `doc/models/containers/{union}.md` lists every case with its factory method.

A **polymorphic** schema is different again: the base class carries a `JsonSubTypes` converter and one
`KnownSubType` row per child, each child sets the discriminator itself, and you narrow the base type
with `is` — the shape is in [reference.md](reference.md).

## Collections, dates and numbers

- List properties are `List<T>` and map properties are `Dictionary<string, V>`. An empty collection is
  serialized; a `null` one follows the optional rule above. Numbers are the declared CLR type — `int`,
  `long` or `double`, and `T?` when optional.
- **A temporal field is only a `DateTime` if the property says so — plenty are plain `string`.** Open
  the property in `Models/{Model}.cs` and read its declared type before writing any conversion. A
  `string` there means the SDK does no date handling at all in either direction: you format and parse it
  yourself, and the wire format is whatever the API documents.
  Where the property *is* temporal it is `DateTime` or `DateTimeOffset` — fixed when the SDK is
  generated, never `DateOnly` — and the wire format is a **per-property converter attribute**, so two
  date fields on one model can travel differently. That attribute table is in
  [reference.md](reference.md), and its absence on a `string` property is the confirmation that nothing
  is converting for you.

## Unknown / future fields

This SDK does **not** carry the extended additional-properties surface, so there is **no indexer** — `m["key"]`
does not compile. Instead, check for `Models/BaseModel.cs`:

- **If it exists**, models are generated as
  `class {Model} : BaseModel`, inheriting one
  `[JsonExtensionData] public Dictionary<string, object> AdditionalProperties { get; set; }`. Read and
  write that property directly. **It is `null` until something assigns a dictionary** — no constructor
  initialises it — so assign one before writing, and null-check before reading a response that carried no
  extra fields. This shape has no collision check against declared `[JsonProperty]` names and no helper
  for typed reads: whatever conversion you need, you write — and note the values come back as **boxed CLR
  primitives**, not `JToken`, so a JSON integer arrives as `long` and `(int)dict["k"]` throws
  `InvalidCastException` where `Convert.ToInt32` works. Reading a key that is not there throws
  `KeyNotFoundException`, so prefer `TryGetValue`. There is also **no collision check** in this shape:
  writing a key that matches a declared `[JsonProperty]` name emits a *second* copy of it, producing a
  duplicate-key document silently. **`: BaseModel` is not a per-model signal** — the generator applies it to
  every root non-exception model on such a build, so it tells you which build you are on, not which models
  accept extras. Nothing in `doc/` distinguishes them on this shape either, so treat "can this model carry
  extras" as unanswerable and just try it.
- **If it does not exist**, no model accepts extra fields and unmodelled JSON is discarded — regenerate
  the SDK or parse that response yourself.

`ls Models/BaseModel.cs` decides which of the two shapes you are in — do that first.

## Reference

The emitted member shapes — plain, optional-and-nullable and immutable models, the three enum
declarations, the union container, the discriminated hierarchy, the additional-properties surface and
the date-converter table — are in [reference.md](reference.md).

## Next

- Step 5, errors and status codes → **csharp-error-handling**
- Back to the call that returned this shape → **csharp-calling-endpoints**
