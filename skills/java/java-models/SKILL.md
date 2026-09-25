---
name: 'java-models'
description: 'Work with models from the PayPal Server SDK Java SDK. Load before building a request payload or mapping a response onto your own types. The class won''t tell you whether models are mutable or builder-only, which annotation argument is the wire name, how a oneOf/anyOf container is matched, or that you never set a discriminator yourself.'
---

# Working with models in an APIMatic Java SDK

Most request/response data are plain generated classes with a nested `Builder` (covered in
`java-calling-endpoints`). This skill covers the **non-obvious model shapes** that trip integrations up.
The patterns are generic across APIMatic Java SDKs; take the real type names from your SDK source under
`<root>/models/`.

## The Builder split: required vs optional

Every model class carries a nested `public static class Builder`. The split is mechanical:

- **Required fields are constructor arguments** on the `Builder`.
- **Optional fields are fluent setters** — and so are the required ones, so you can also override a
  constructor value later.
- `build()` returns the model; `model.toBuilder()` returns a `Builder` pre-populated from an existing
  instance, which is how you derive a variant without re-listing every field.

```java
{Model} m = new {Model}.Builder(requiredA, requiredB)
        .optionalC(value)
        .build();

{Model} variant = m.toBuilder().optionalC(other).build();
```

A model with **no** required fields has no `Builder` constructor arguments at all — start from
`new {Model}.Builder()`. A model that extends another names its clone method after itself
(`to{Model}Builder()`) instead of `toBuilder()`.

**This SDK's models are mutable**, so each field also has a plain `@JsonSetter` setter
and the model has a no-arg constructor alongside the public all-args one. A model with required fields
additionally gets a **no-arg `Builder()` overload**, so `new {Model}.Builder()` compiles even there and
the required values can be supplied with the fluent setters instead — the `Builder` constructor is the
form that keeps them checked.

## Optional vs nullable-optional — the `unset` case

Two different things get called "optional":

- **Optional** — a plain boxed type (`String`, `Integer`, `Boolean`, …). Leave it unset and it is omitted
  from the JSON: the getter carries `@JsonInclude(JsonInclude.Include.NON_NULL)`.
- **Optional *and* nullable** — the field can be *absent* or explicitly `null`, which are different on
  the wire. The private field is an `OptionalNullable<T>`, and the model gains a **fourth** member:

```java
{Model} m = new {Model}.Builder()
        .{field}(null)     // send an explicit  "{field}": null
        .build();

{Model} n = new {Model}.Builder()
        .{field}(value)
        .unset{Field}()    // omit "{field}" entirely
        .build();
```

The getter still returns the plain `T` (`get{Field}()`); `unset{Field}()` exists on both the model and the
`Builder`, so you can also clear the field after the fact. If you see an `unset{Field}()` method, that
field distinguishes absent from null — and `.{field}(null)` is *not* the same as leaving it alone.

## Enums

Generated enums are real Java `enum` types with a string (or integer) wire value attached. **Do not use
`valueOf`** — that resolves the Java constant identifier, which is an upper-cased, sanitized version of
the name and is often not the wire value. Use the generated factories:

```java
{EnumType} e = {EnumType}.{CONSTANT};              // known constant
{EnumType} f = {EnumType}.fromString("wire_value"); // unmapped -> _UNKNOWN, or null; see below
String wire  = e.value();                          // back to the wire value
```

Integer-backed enums use `fromInteger(Integer)` and `Integer value()` instead. Both kinds also expose
`constructFromString`/`constructFromInteger` (the Jackson entry point, which throws `IOException` only
where the enum has no `_UNKNOWN` member — see below — on
an unmapped value) and a `toValue(List<{EnumType}>)` helper for converting a whole list.

**Most enums carry an extra `_UNKNOWN` member** — it is generated wherever the API allows values outside
the declared set, which in practice is nearly all of them. Check yours rather than assuming either way:
`for f in <root>/models/*.java; do grep -q "public enum" $f && ! grep -q _UNKNOWN $f && echo $f; done`
lists the exceptions, and it is often a very short list.

Where `_UNKNOWN` exists it changes two things the surrounding prose would otherwise imply:

- **`fromString` returns `_UNKNOWN`, never `null`**, for anything it does not recognise, and
  `_UNKNOWN.value()` is `null`. A null check will not detect an unmapped value.
- **`constructFromString` cannot throw for it either.** That method delegates to `fromString` and throws
  only on a `null` return, so on an `_UNKNOWN`-carrying enum the throw is unreachable — catching
  `IOException` to detect an unmapped value silently gets `_UNKNOWN` instead.

Both matter before you write an exhaustive `switch`, and both invert on the minority of enums that lack
the member.

## oneOf / anyOf union types

**Check this SDK has any before reading on:** `ls <root>/models/containers/`. No such package means
the API declares no `oneOf`/`anyOf` type, none of the container classes, `match` methods or `Cases`
interfaces below are generated, and an import of them will not compile. Skip to the next section.

When a field can be one of several types, APIMatic generates an **abstract container class** under
`<root>/models/containers/`.

### Construct — a static factory, never a constructor

The case classes are private; the only way in is a static `from{Variant}(...)` factory on the container:

```java
import {rootPackage}.models.containers.{Union};

{Union} u = {Union}.from{Variant}(variantValue);
```

### Read — `match` with a `Cases<R>` visitor

The container declares `public abstract <R> R match(Cases<R> cases);` and a nested
`public interface Cases<R>` with one method per variant. That visitor is the exhaustive way to unwrap it:

```java
String described = u.match(new {Union}.Cases<String>() {
    @Override
    public String {variantA}({VariantAType} value) {
        return "A: " + value;
    }

    @Override
    public String {variantB}({VariantBType} value) {
        return "B: " + value;
    }
});
```

Open the container file and read its `Cases` interface for the real variant method names and types —
they come from the spec, not from a fixed convention. A discriminated union additionally registers its
discriminator values in the container's deserializer, so the right case is selected automatically on the
way in.

## Collections, dates and numbers

- List properties are `java.util.List<T>`; map properties are `java.util.Map<String, V>`. Assign an
  ordinary `Arrays.asList(...)` / `HashMap`. A `null` collection is omitted from the JSON when the
  property is **optional** (its getter carries `@JsonInclude(NON_NULL)`); a **required** collection has
  no such annotation, so `null` is written as an explicit `null`. An empty collection is always
  serialized.
- **Check whether this SDK has date types at all before assuming one:**
  `grep -rl "java.time" <root>/models/`. No hits means **every temporal field is a `String`** — read and
  written verbatim, with no conversion and no validation in either direction, and you format it
  yourself. That is a common outcome and it is not a degraded one; it simply follows from how the API
  declared those fields.
  Where the grep *does* hit, the fields are real `java.time` types: `LocalDate` for a plain date, and
  either `LocalDateTime` or `ZonedDateTime` for a date-time depending on whether timezone information is
  kept. **`OffsetDateTime` is never generated** — convert at your boundary if your own code uses it. The
  wire format is then handled by a generated `DateTimeHelper` through Jackson annotations. If there is
  no `DateTimeHelper` under `<root>/utilities/`, there is no conversion layer.
- Numeric and boolean fields are Java primitives (`long`, `int`, `double`, `boolean`) when required and
  non-nullable, and boxed (`Long`, `Integer`, `Double`, `Boolean`) as soon as they are optional or
  nullable — precisely so that `null` is expressible.

## Polymorphic (discriminated) hierarchies

**Check this SDK has any before reading on:** `grep -rl '@JsonTypeInfo' <root>/models/`. No hits means
no model has subtypes, every generated model is a plain class extending nothing, and the `instanceof`
narrowing below has nothing to apply to.

A model with subtypes carries Jackson's `@JsonTypeInfo` / `@JsonSubTypes` at the top of the class, and
each child's own constructor and `Builder` pre-set its discriminator value, so you never assign it.
Deserialization picks the concrete subclass for you — a response field typed as the parent may
therefore hold a child at runtime. Use
`instanceof` (or read the discriminator getter) to narrow, and construct the **child** class directly
when sending one.

## Unknown / future fields

By default a model declares its properties explicitly and unknown JSON fields are dropped on
deserialization. Two opt-in shapes preserve them, and which one an SDK uses is fixed at generation time:

- the model `extends BaseModel`, inheriting a `getAdditionalProperties()` map; or
- the model holds its own `AdditionalProperties` field and exposes a public
  `getAdditionalProperty(String name)` — the map itself stays private, wired to Jackson via
  `@JsonAnyGetter`/`@JsonAnySetter`, and the `Builder` gains `additionalProperty(String name, T value)`
  for the write side.

Check the model class for either a base class or an `additionalProperty` member. If neither is present,
an unmodelled field cannot be read through the SDK — regenerate it or parse that response yourself.

See [reference.md](reference.md) for the exact emitted member shapes.

## Next

- Step 5, errors and status codes → **java-error-handling**
- Back to the call that returned this shape → **java-calling-endpoints**
