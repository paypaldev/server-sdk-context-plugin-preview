---
name: 'php-models'
description: 'Work with models from the PayPal Server SDK PHP SDK. Load before building a request payload or mapping a response onto your own types. The class won''t tell you the setter docblocks drive deserialization at runtime, how absent, null and set are told apart, or which builder method takes a field back off the wire.'
---

# Working with models in an APIMatic PHP SDK

Models are plain PHP classes in `src/Models/`: private fields, a constructor taking the **required**
fields, `get{Field}()` / `set{Field}()` pairs, and `jsonSerialize()`. Beside each one, in
`src/Models/Builders/`, is a `{Model}Builder`. This skill covers the shapes that trip integrations up;
the plain scalar case is in **php-calling-endpoints**.

## Building: the builder's `init(...)` is the required-field list

```php
use PaypalServerSdkLib\Models\Builders\{Model}Builder;

$model = {Model}Builder::init(/* required fields, positionally */)
    ->{optionalField}($value)
    ->build();
```

`build()` returns a **clone**, so a builder can be reused. `new {Model}(...)` plus `set{Field}(...)` is
the equivalent longhand.

## PHP name vs JSON name

The setter's docblock carries `@maps {jsonName}` — that, not the method name, is the wire field. They
diverge whenever the JSON name collides with a PHP keyword or an inherited member, so a JSON `code`
field can surface as `getCodeProperty()` / `setCodeProperty()`. When you need the wire name, read the
`@maps` annotation (or the model's `jsonSerialize()`), not the getter.

## Enums — a class of constants, not a PHP `enum`

These SDKs target PHP 7.2+, so an enum is generated as a **plain class** holding `public const` values.
Whether that class *also* carries a static `checkValue()` validator, and whether any setter names it, is
what varies:

```php
use PaypalServerSdkLib\Models\{EnumClass};

$model = {Model}Builder::init({EnumClass}::{CONSTANT})->build();
```

**This SDK validates its closed enums.** A **closed** enum — one whose definition does not
accept values outside the listed set — gets `checkValue()` beside its constants, and an **open** enum gets
the constants and nothing else. Closed vs open is decided **per enum**, from that enum's own definition,
not once per SDK. One SDK routinely carries both kinds side by side — a large API can come out 85 open
enums to one closed — so a `checkValue` hit on one class tells you nothing about the next. Check the class
you are about to rely on, `grep -l checkValue src/Models/{EnumClass}.php`, rather than the directory.

Three consequences:

- **The field and parameter type is `string` (or `int`), not the enum class.** A signature saying
  `string $status` is still an enum field. For a **closed** enum the setter's docblock names the class
  (`@factory …{EnumClass}::checkValue`), and passing an unlisted string is accepted by PHP then rejected
  at (de)serialization time with an `Exception` reading `"<value> is invalid for {EnumClass}."`. For an
  **open** enum there is no annotation at all and an unlisted string goes out on the wire — `doc/models/`
  and the operation's parameters table are then the only places the enum class is named.
- **Constant names are derived from the values and are often mangled** — a trailing underscore to dodge
  a PHP keyword, underscores inserted mid-word. Never guess the constant; open `src/Models/{EnumClass}.php`
  and read it.
- `{EnumClass}::checkValue($value)` is public **when it exists** — only closed enums get one, so check the class before calling it.

`Environment` and `Server` in `src/` use the same class-of-constants shape but have **no** `checkValue()`.

## oneOf / anyOf unions — no container class

Unlike some APIMatic languages, PHP generates **no union container type**, no `from{Variant}()` factory
and no visitor. A union field is:

- typed only in the docblock — `@var Atom|Orbit` — and frequently has **no PHP type declaration at all**
  on the getter/setter, so nothing checks it at assignment;
- annotated on the setter with `@mapsBy anyOf(Atom,Orbit)` (or `oneOf(...)`, nested and combined with
  `null` for nullable cases), which is the template the runtime validates against.

**Write** by assigning the variant directly:

```php
$model = {Model}Builder::init(new Atom(/* … */))->build();
```

**Read** by type-testing yourself — this is the only mechanism:

```php
$value = $response->get{Field}();
if ($value instanceof Atom) {
    // …
} elseif ($value instanceof Orbit) {
    // …
}
```

For scalar members use `is_string()` / `is_int()` / `is_float()` rather than `instanceof`.

Validation happens when the value crosses the wire, not when you set it — so a wrong variant surfaces as
a serialization exception on the call, with the `@mapsBy` template in the message. Read the setter's
`@mapsBy` annotation (and `doc/models/containers/*.md`, which documents the cases) before debugging your
own input.

## Optional vs nullable — the `unset{Field}()` distinction

A field that is **both optional and nullable** gets a third method:

```php
$model->set{Field}(null);    // send  "{field}": null
$model->unset{Field}();      // omit "{field}" from the payload entirely
$model->get{Field}();        // null in BOTH cases — the getter cannot tell them apart
```

The builder mirrors it with `->unset{Field}()`. If `unset{Field}()` exists on the model, that field
distinguishes "absent" from "explicitly null" and you must pick deliberately. If it does not, the field
is plain-optional: leaving it unset omits it, and there is no way to send an explicit `null`.

## Collections, dates and numbers

- **Lists and maps are both plain PHP `array`.** The docblock distinguishes them (`{Type}[]` vs
  `array<string,{Type}>`); the type declaration says only `array`.
- **A temporal field is `\DateTime` only where the model says so; otherwise it is `?string`.** Read the
  declared type on the property in `src/Models/{Model}.php` first. `?string` means the SDK converts
  nothing in either direction — you format and parse it yourself. A quick whole-SDK check is
  `grep -rl '\DateTime' src/Models/`; no hits means no field is temporal, and
  `src/Utils/DateTimeHelper.php` will not exist either.
  Where a field *is* `\DateTime`, pass one and the SDK converts, and the `@factory` annotation on the
  setter names the exact `DateTimeHelper` function — which is how you tell which wire format that field
  uses.
- Numbers are `int` or `float` per the model; money and identifiers may be `string` — the declared type
  is the source of truth.

## Unknown / future fields

A model keeps only the fields it declares. Unknown JSON is **dropped** on deserialization unless the
model carries additional-properties support, which shows up as
`addAdditionalProperty(string $name, $value)` and `findAdditionalProperty(string $name)` on the class:

```php
$extra = $model->findAdditionalProperty('someKey');   // returns FALSE when absent, not null
if ($extra !== false) {
    // …
}

$model = {Model}Builder::init(/* … */)
    ->additionalProperty('someKey', $value)           // note: singular, unlike the model's addAdditionalProperty
    ->build();
```

`findAdditionalProperty` returning `false` rather than `null` matters — a `?? ` or `is_null()` check
silently misses it. If the class has neither method, regenerate the SDK or parse that response yourself.

## Reference

The enum class shape, the full `@mapsBy` template grammar, the `DateTimeHelper` function families and
the additional-properties details are in [reference.md](reference.md).

## Next

- Step 5, errors and status codes → **php-error-handling**
- Back to the call that returned this shape → **php-calling-endpoints**
