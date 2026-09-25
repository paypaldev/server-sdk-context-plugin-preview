# Models reference (APIMatic C#)

The exact member shapes the C# generator emits, so you can recognise each one on sight in `Models/`.
The shapes are the same in every SDK; only the names are generated. Which shape a given class uses is
**fixed when the SDK is generated** — read the class, do not infer it from another model.

## Plain model — the default shape

```csharp
public {Model}() { }

public {Model}(
    string {requiredField},                     // required — positional, assigned unconditionally
    string {optionalField} = null)              // optional — C# optional argument
{
    this.{RequiredField} = {requiredField};
    this.{OptionalField} = {optionalField};     // plain optional — assigned unconditionally too
}

[JsonProperty("{requiredField}")]
public string {RequiredField} { get; set; }

[JsonProperty("{optionalField}", NullValueHandling = NullValueHandling.Ignore)]
public string {OptionalField} { get; set; }
```

The `[JsonProperty]` argument is the **wire name** and wins over the C# property name, which is usually
a PascalCase rendering of it — but a rendering that lands on a C# keyword or reserved word is prefixed
with `M` (wire `value` → `MValue`, `string` → `MString`, `Event` → `MEvent`), so read the property name
off the class rather than deriving it. A required property carries no `NullValueHandling` flag and **nothing
enforces it** — an unset one serializes as JSON `null`, or as its `default` for a value type. Every
model also gets a `ToString()` printing `{Model} : (Field = value, …)`; that is **not** JSON, so use
`ApiHelper.JsonSerialize(...)` from `PaypalServerSdk.Standard.Utilities` when you need the wire form.

## Optional-and-nullable field

A field that is both optional *and* explicitly nullable drops the auto-property for a hand-written one
over a private backing field, plus a `shouldSerialize` map:

```csharp
private string {field};
private Dictionary<string, bool> shouldSerialize = new Dictionary<string, bool>
{
    { "{field}", false },
};

[JsonProperty("{field}")]                                                   // no NullValueHandling
public string {Field}
{
    get { return this.{field}; }
    set { this.shouldSerialize["{field}"] = true; this.{field} = value; }   // any assignment arms it
}

public void Unset{Field}() { this.shouldSerialize["{field}"] = false; }
public bool ShouldSerialize{Field}() { return this.shouldSerialize["{field}"]; }
```

The `Unset{Field}()` + `ShouldSerialize{Field}()` pair is the reliable way to spot the shape. Exception
classes never get it.

## Immutable model shape

Generated only when the SDK was built with immutable models on. Properties become `{ get; }`-only,
the parameterless constructor disappears — so an object initializer will not compile — and a public
nested `Builder` appears instead:

```csharp
public Builder ToBuilder() { … }

public class Builder
{
    public Builder(string {requiredField}) { … }        // required fields only
    public Builder {RequiredField}(string value) { … }  // one fluent setter per field
    public Builder Unset{Field}() { … }                 // optional-and-nullable fields only
    public {Model} Build() { … }
}
```

Collection properties become `ImmutableList<T>` / `ImmutableDictionary<string, V>`.

## String-backed enum

```csharp
[JsonConverter(typeof(StringEnumConverter))]        // APIMatic.Core.Utilities.Converters
public enum {EnumType}
{
    [EnumMember(Value = "in_transit")]
    InTransit,
}
```

## Integer-backed enum

```csharp
[JsonConverter(typeof(NumberEnumConverter))]
public enum {EnumType}
{
    {Member} = 3,                                   // no [EnumMember] at all
}
```

## Unknown-tolerant enum

When the spec allows values beyond the declared set the converter is wrapped and one member is added:

```csharp
[JsonConverter(typeof(UnknownEnumConverter<StringEnumConverter>), nameof(_Unknown))]
public enum {EnumType}
{
    [EnumMember(Value = "in_transit")] InTransit,
    _Unknown,                                       // anything unrecognised lands here
}
```

`_Unknown` is the one member emitted **without** an `[EnumMember]`, so it has no wire value — and
serializing it **throws**: `ApiHelper.JsonSerialize({EnumType}._Unknown)` raises
`JsonSerializationException: ... is not a valid enum value for serialization!`. The value is therefore
receive-only: an unrecognised wire string deserializes into it safely, but sending it back is a runtime
failure, not a lossy round trip. Branch on it before you serialize anything that might hold it. A `switch` over the
declared members alone misses it. Check for the wrapped converter before writing an exhaustive one.

## oneOf / anyOf container

A container in `PaypalServerSdk.Standard.Models.Containers` is an abstract class carrying a
`[JsonConverter(typeof(UnionTypeConverter<{Union}>), …)]` attribute whose trailing bool is `true` for
`oneOf` and `false` for `anyOf`.

**Count the attribute's arguments — a container union can be discriminated, and that changes how it
matches.** The three-argument form matches **structurally**: the body is tried against each case, and for
`oneOf` matching none or more than one throws. What makes that match decidable is **`[JsonRequired]` on the
variants' required properties** — where a container's variants are object models, grep `Models/` for it and
you will find it on those variants and nowhere else. (A container whose cases are bare scalars carries none,
because there is nothing to discriminate.) **Two variants with identical `[JsonRequired]` sets cannot be
told apart**: every input matches both, so a `oneOf` throws on every payload and the container is
undeserializable by construction. If a union fails on input you believe is valid, compare the variants'
required sets before doubting your payload. So a body missing one of a variant's required fields does not match it, and a body carrying
the required fields of *two* variants matches both and throws on a `oneOf`. Unmodelled extra fields do
**not** affect the match; they land in that variant's own additional-properties surface.

A **five-argument** form adds a discriminator: one more array, giving one value per case in case order,
plus a string naming the field that carries it. The trailing bool means the same thing here as in the 3-argument form, and a discriminated container can be
**either**. Check it before writing the catch: a discriminated `anyOf` (trailing `false`) raises
`AnyOfValidationException`, not `OneOfValidationException`.

Here the named field is `[JsonRequired]` on every variant and must equal one of the listed values. Three
consequences, all of which bite:

- **`From{Variant}(...)` does not set the discriminator for you.** Assign it on the variant yourself, to the
  literal from the attribute — and how you find out you forgot depends on the field's type, so check which
  you have:
  - a **`string`** discriminator is `[JsonRequired]`, so **serialization throws**:
    `JsonSerializationException: Cannot write a null value for property '{discriminatorField}'. Property
    requires a value.`
  - an **enum** discriminator is a non-nullable value type, so there is nothing to throw about:
    `default(TEnum)` is member 0, whose literal belongs to *some other case*. The payload serializes happily
    and **round-trips back as the wrong variant, silently**. This is the worse of the two, and no exception
    marks it.
- **A value outside the list throws `OneOfValidationException`** on the way in, and the match is
  case-sensitive: a correctly-spelled-but-differently-cased value fails too.
- **Matching is by discriminator alone, not by shape.** Where the variants' only `[JsonRequired]` field is
  the discriminator, a body carrying *one* variant's fields under *another* variant's label binds to the
  label's variant and **silently drops the fields that do not belong to it**. There is no error. If a
  round-trip loses data, suspect a mislabelled discriminator before suspecting the fields.

**Read the attribute before constructing or parsing one**, and take the values from it rather than from the
variants' enum type, which may admit more.

You build one with `{Union}.From{Variant}(value)` and read it with `Match<T>(...)`, which takes one
callback per variant. The case classes are **`private sealed`** — a type you cannot name, so there is no
`switch`, no cast, and no `Is{Variant}`/`Get{Variant}` accessor. `MatchSome` is `Match` with optional
callbacks, and a case whose callback is `null` yields `default(T)` silently.
`doc/models/containers/{union}.md` lists every case with its factory.

## Discriminated hierarchy

A polymorphic schema emits the base class with a `JsonSubTypes` converter naming the discriminator
property and one `KnownSubType` row per child:

```csharp
[JsonConverter(typeof(JsonSubtypes), "{discriminatorField}")]
[JsonSubtypes.KnownSubType(typeof({ChildA}), "{childAValue}")]
[JsonSubtypes.KnownSubType(typeof({ChildB}), "{childBValue}")]
public class {Parent}
```

Each child sets the discriminator itself, so you never assign it by hand — construct the concrete
child and read the response back as the base type, narrowing with `is`. **Which values map to which
child is the `KnownSubType` list**, so read it rather than the spec.

## Additional (unknown / future) properties

Support is **not** per-model, and the generator has **two different shapes** for it. Which one a build
carries varies per SDK, and one file tells you which — so `ls` before writing any access:

| What you find | Which shape |
| --- | --- |
| `Models/BaseModel.cs` | a shared base class the accepting models inherit, exposing a **public dictionary property** |
| `Utilities/AdditionalPropertiesExtensions.cs`, and **no** `Models/BaseModel.cs` | a per-model private `[JsonExtensionData]` dictionary behind a **`public this[string key]` indexer on the model itself** |
| neither file | no model in this SDK accepts extra fields — unmodelled JSON is silently dropped |

The two are mutually exclusive: the extended setting suppresses `BaseModel.cs` entirely.

### Shape 1 — the shared `BaseModel`

```csharp
// Models/BaseModel.cs — generated only when the API uses additionalProperties
public class BaseModel
{
    [JsonExtensionData]
    public Dictionary<string, object> AdditionalProperties { get; set; }
}

// and the models that accept them:
public class {Model} : BaseModel { … }
```

So the surface here is a **public dictionary property**, read and written directly —
`model.AdditionalProperties["someKey"]` — and there is no indexer on the model.

> **In this shape the property has no initializer and no constructor assigns it**, so on a model you
> build yourself it is `null` and `model.AdditionalProperties["someKey"] = x` throws
> `NullReferenceException`. Assign a dictionary first:
> `new {Model} { AdditionalProperties = new Dictionary<string, object>() }`. Only a model Newtonsoft
> deserialized comes back with one — and even then only when the JSON carried extra fields, so
> null-check before reading too.

**Do not treat `: BaseModel` as a per-model signal.** On a build that emits `BaseModel.cs`, the generator
applies the base class to **every** root non-exception model regardless of whether that model's schema
allows extra fields, so grepping one model for `: BaseModel` tells you which *build* you are on and nothing
about which models accept extras. The per-model answer is in `doc/models/{model}.md`, which prints an
"accepts additional fields" line only for the models that do.

### Shape 2 — the per-model indexer

```csharp
// Models/{Model}.cs on a build with the extended surface — no BaseModel involved
[JsonExtensionData]
private readonly IDictionary<string, JToken> additionalProperties;   // constructor-initialised

[IndexerName("AdditionalPropertiesIndexer")]
public {T} this[string key]
{
    get => additionalProperties.GetValue<{T}>(key);
    set => additionalProperties.SetValue(key, value, propertyName);
}
```

The dictionary is **not** reachable — the indexer is the whole surface, `{T}` is the declared
additional-fields type (`object` unless the schema pins it), and the setter rejects a key that collides
with one of the model's declared **JSON wire names**, not its C# property names: on a model with
`[JsonProperty("line1")] public string Line1`, `m["line1"]` throws `ArgumentException` while `m["Line1"]`
is accepted. Because the constructor initialises the dictionary, there is nothing to null-check and
nothing to assign first — **on a model**. Typed exceptions carry the same indexer over a dictionary that is
never initialised, so `e["anything"]` throws `NullReferenceException` on read and `ArgumentNullException`
on write; that is the same defect behind the off-shape error body in **csharp-error-handling**.

> **`KeyNotFoundException` does not always mean the key is missing.** The typed read swallows a
> deserialization failure and rethrows it as `KeyNotFoundException`, so a key that *is* present but does
> not convert to `{T}` reports as absent. This only bites where the indexer's declared type is narrower
> than `object`; if a key you know you wrote reads back as missing, suspect the type before the key. One variation worth reading off the class rather than assuming:
on an immutable-models build the indexer's `set` is `private` — values go in through the model's
`Builder.AdditionalProperty(key, value)` instead — and on that build a model that has subclasses
declares the field `protected` rather than `private`. Here the signal is the indexer, not a base class,
and `doc/models/{model}.md` also carries *"This model accepts additional fields of type …"* — a line
the `BaseModel` shape never prints. For typed reads, `.ToObject<{T}>()` comes from
`Utilities/AdditionalPropertiesExtensions.cs`, the file whose presence marks this shape. The SKILL
resolves which of the two shapes **this** SDK has, so start there rather than inferring it here.

## Dates and times

The CLR type is fixed at generation; the wire format is a **per-property converter attribute**.

| Attribute | Where it lives | Wire format |
| --- | --- | --- |
| `[JsonConverter(typeof(IsoDateTimeConverter))]` | Newtonsoft | full ISO-8601 date-time |
| `[JsonConverter(typeof(CustomDateTimeConverter), "yyyy'-'MM'-'dd")]` | `PaypalServerSdk.Standard.Utilities` | the literal format string given |
| `[JsonConverter(typeof(UnixDateTimeConverter))]` | `PaypalServerSdk.Standard.Utilities` | Unix timestamp |

Operation *parameters* carry no converter — the controller formats those inline with `.ToString(...)`,
per parameter.

## Notes

1. `Models/{Model}.cs` — the class file is the **source of truth** for the shape, the wire names, the
   required/optional split and the converters.
2. `Models/Containers/` exists only when the API declares a `oneOf`/`anyOf`, and
   `Utilities/AdditionalPropertiesExtensions.cs` only when extended additional-properties support was
   on. **Their absence is meaningful**: `ls` the directory rather than assume.
3. `doc/models/{model}.md` — the same fields with a **Tags** column marking the required ones. On a
   `BaseModel` build it says **nothing** about additional properties, so `: BaseModel` on the class is
   the only signal; on a build with the extended surface it *does* print *"This model
   accepts additional fields of type …"*, and the signal on the class is the `this[string key]`
   indexer.
4. `Equals` is emitted on every model in some builds and on none in others (exception classes never
   get it). **`GetHashCode` is emitted only on immutable-models builds**, so on an ordinary build a
   model compares by value and still hashes by
   reference identity — it is broken as a `Dictionary`/`HashSet` key even though the `Equals` grep
   succeeds.
