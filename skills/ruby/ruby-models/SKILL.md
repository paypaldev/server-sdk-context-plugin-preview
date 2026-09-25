---
name: 'ruby-models'
description: 'Work with models from the PayPal Server SDK Ruby SDK. Load before building a request payload or mapping a response onto your own types. The class won''t tell you how omitted and nil are told apart, which attribute names reach the wire, how a union is resolved, or what a model-free build gives you instead.'
---

# Working with models in an APIMatic Ruby SDK

Models live in `lib/paypal_server_sdk/models/`, one class per model, each extending
`PaypalServerSdk::BaseModel`. Most are plain data objects (covered in `ruby-calling-endpoints`); this
skill covers the shapes that trip integrations up.

## Anatomy of a generated model

Every model class declares the same members, and reading them answers most questions faster than
guessing:

| Member | What it tells you |
| --- | --- |
| `attr_accessor :{attribute}` | the Ruby attribute names, snake_cased from the spec |
| `self.names` | the map from Ruby attribute name → **JSON wire name**. The two differ whenever the spec's name isn't snake_case |
| `self.optionals` | which attributes may be omitted |
| `self.nullables` | which attributes may be sent as JSON `null` |
| `initialize` | the construction shape — see below |
| `self.from_hash(hash)` | deserialization; the SDK calls it for you |
| `to_hash` / `to_json` | serialization, inherited from `BaseModel` |
| `to_s` / `inspect` | a readable dump of every attribute — the quickest way to see what you actually built |

`initialize` comes in **one of two shapes, fixed when the SDK is generated**: positional arguments each
with a default, or keyword arguments. **Open the model and read it** — do not copy the call style from another
SDK:

```ruby
# positional form
PaypalServerSdk::{Model}.new({required_value}, {optional_value})

# keyword form
PaypalServerSdk::{Model}.new({required_attr}: {required_value}, {optional_attr}: {optional_value})
```

## Omitted vs. nil — the rule that decides what reaches the wire

An optional attribute defaults to a private `SKIP` sentinel, and `initialize` only assigns the instance
variable when the value isn't `SKIP`. `to_hash` walks the model's **instance variables**, so:

- **Omit an optional attribute** → its instance variable is never set → the key is absent from the JSON.
- **Pass `nil` explicitly** → the key is still dropped, *unless* the attribute is listed in
  `self.nullables`, in which case it is sent as JSON `null`.
- **A required attribute you leave `nil`** is dropped too. Nothing in `to_hash` raises for it, so a
  request missing a required field fails at the **API**, with a 4xx, not in Ruby. If a call returns an
  unexpected 400/422, dump the model with `to_s` and compare against `self.names` before suspecting the
  SDK.

Setting an attribute after construction works normally (`model.{attribute} = value`) and does set the
instance variable, so it is a valid way to fill in an optional attribute you skipped.

## Enums

An enum is a **class of constants**, not a Ruby symbol and not a Ruby `Enum`. Most hold frozen strings;
an enum the spec declares as integer-based holds plain `Integer`s instead — only the enclosing array is
frozen, and `doc/models/{enum_type}.md` calls it "A integer based enum":

```ruby
module PaypalServerSdk
  class {EnumType}
    {ENUM_TYPE} = [
      {SOME_CONSTANT} = 'someValue'.freeze,   # integer-based enum: {SOME_CONSTANT} = 1
      {OTHER_CONSTANT} = 'otherValue'.freeze
    ].freeze
  end
end
```

Plus two class methods: `self.validate(value)`, which answers whether a value is a member, and
`self.from_value(value, default_value = {SOME_CONSTANT})`, a lenient lookup.

**Pass the constant.** For a string enum the constant and the raw string are the same object, so either
works; for an integer enum they are not interchangeable — the constant holds a number, `validate` does an
`include?` against numbers, and assigning `'{SOME_CONSTANT}'` or `'1'` puts a string on the wire where
the API expects an integer:

```ruby
model.{enum_attribute} = PaypalServerSdk::{EnumType}::{SOME_CONSTANT}   # always correct
model.{enum_attribute} = 'server_provided_value'   # string enums only; nothing validates on assignment
```

**Open the enum's file** and look at what the constants are assigned before writing a raw value.
`from_value` is not the escape hatch it looks like: its `case` arms match the **constant names**
downcased, not the wire values, so `from_value('05')` on an enum whose member is `ENUM_05 = '05'` matches
nothing (an integer enum adds one numeric-string arm, and that is the only exception). It then falls back
to the first constant rather than raising, so a value it did not recognise reaches the wire as a
different, valid-looking one. Read its generated `case` arms before feeding it anything, and do not use
it to reject bad input. An enum generated from a spec that allows additional values has a `validate` that
returns `true` for everything.

## oneOf / anyOf union attributes

**Check this SDK has any before reading on:** `ls lib/paypal_server_sdk/utilities/`. No
`union_type_lookup.rb` there means the API declares no `oneOf`/`anyOf` type and nothing below
applies. Skip to the next section.

A union attribute is **plain Ruby**: there is no wrapper class and no factory. Assign whichever variant
value you have — a model instance, a string, a number, an array:

```ruby
model.{union_attribute} = PaypalServerSdk::{VariantModel}.new(...)
model.{union_attribute} = 'a plain string'
```

The generated machinery lives elsewhere:

- `lib/paypal_server_sdk/utilities/union_type_lookup.rb` holds `UnionTypeLookUp.union_types`, a hash of
  every union in the SDK keyed by name. **This is where you read a union's accepted variants**, its
  discriminator, and whether it is oneOf or anyOf.
- `from_hash` resolves an incoming value with
  `APIHelper.deserialize_union_type(UnionTypeLookUp.get(:{UnionName}), hash['{wire_name}'])`. On the way
  out, `BaseModel#to_hash` calls a `to_union_type_{attribute}` hook **if the model defines one** — the
  generator emits that method only for a union that contains a date/time case, so that the date is written
  in its declared format. For every other union no such method exists and the value serializes through the
  normal path (`value.to_hash` for a model, the raw value for a scalar). Check the model before going
  looking for it.
- A model that *is* a union case, or that has a union attribute, also gets `self.validate(value)`, which
  `to_hash` invokes before serializing.

The consequence to plan for: **a union is validated when the model is serialized, not when you assign
it**, so a value that doesn't satisfy the union surfaces at the point of the call rather than where you
set it. Read the union's entry in `union_type_lookup.rb` and the variant models' required attributes
before debugging your own input; `doc/models/` documents each union's accepted types in human-readable
form.

## Discriminated inheritance

When a spec uses a discriminator, the base model gains `self.discriminators` — a map from discriminator
value to child class — and its `from_hash` delegates to the right child automatically. You get the child
instance back, so branch with `is_a?` or read the discriminator attribute. Constructing one means
instantiating the **child** class directly; its `initialize` sets the discriminator for you.

## Collections

Array attributes are Ruby `Array`s of the element type; map attributes are `Hash`es keyed by string.
Assign them directly:

```ruby
model.{list_attribute} = ['A', 'B']
model.{map_attribute} = { 'key' => 'value' }
```

`to_hash` recurses into arrays and hashes, calling `to_hash` on any nested model. An empty array or hash
is serialized; a `nil` one follows the omitted-vs-nil rule above.

## Dates and times

**Check first whether this SDK converts dates at all:**
`grep -rl to_custom_ lib/paypal_server_sdk/models/`. No hits means **every temporal attribute is a plain
`String`** in both directions — no parsing on the way in, no re-formatting on the way out. Assigning a
`DateTime` to one of them serialises it through Ruby's default `to_s`, which is not the RFC 3339 form
most APIs expect, and nothing will warn you. Format and parse the string yourself.

Where the grep *does* hit, those attributes are Ruby `DateTime` objects: `from_hash` parses them with
`DateTimeHelper.from_*`, and the matching `to_custom_{attribute}` re-formats them in the wire format the
API declared — RFC 3339, RFC 1123 or a Unix timestamp. There you pass and receive `DateTime` and the SDK
handles the string form.

**Date-only** attributes may be left as raw strings. Where the model has no `to_custom_{attribute}` and
`from_hash` reads the key directly with no parse, you receive a `String` and must call `Date.iso8601`
yourself — even though `doc/models/` types the attribute `Date` and its example passes a `Date`. **Grep
the model for `to_custom_`** before assuming any conversion exists in either direction.

`PaypalServerSdk::DateTimeHelper` (`lib/paypal_server_sdk/utilities/date_time_helper.rb`, documented in
`doc/date-time-helper.md`) exposes the conversions in both directions — `from_rfc3339`, `from_rfc1123`,
`from_unix`, `to_rfc3339`, `to_rfc1123`, `to_unix`, plus `_map`/`_array` variants — for when you need to
convert values yourself. `PaypalServerSdk::APIHelper.rfc3339` parses an RFC 3339 string safely.

## File uploads

A file parameter is a `PaypalServerSdk::FileWrapper`
(`lib/paypal_server_sdk/utilities/file_wrapper.rb`), which pairs the file with its content type:

```ruby
PaypalServerSdk::FileWrapper.new(File.open('report.pdf'), content_type: 'application/pdf')
```

`content_type` defaults to `'application/octet-stream'` — set it when the API cares.

## Unknown / future fields

By default a model declares its attributes explicitly and `from_hash` reads only those, so an unmodelled
JSON field is dropped on deserialization. Where a model supports additional
properties, `initialize` takes a final `additional_properties` hash and **spreads each key into its own
instance variable** — there is no bag object and no `additional_properties` reader, so read them back
with `get_additional_properties`, inherited from `BaseModel`. `to_hash` writes them out alongside the
declared attributes, under their own names. Nothing raises when one of those keys collides with a
declared attribute: the spread runs *before* the declared assignments, so the declared value silently
wins — and if you omitted that attribute, the extra silently takes its place and is serialized under its
wire name. **Check the model's `initialize` for an `additional_properties` parameter**; if it isn't
there, reading an unmodelled field means regenerating the SDK or parsing that response yourself.

## Next

- Step 5, errors and status codes → **ruby-error-handling**
- Back to the call that returned this shape → **ruby-calling-endpoints**
