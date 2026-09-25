# Models reference (APIMatic PHP)

## Model class shape

A model is a `\JsonSerializable` class with private fields, a constructor taking the required ones,
`get{Field}()`/`set{Field}()` pairs, `__toString()` (via `ApiHelper::stringify`) and
`jsonSerialize(bool $asArrayWhenEmpty = false)`.

The docblock on each setter is **load-bearing at runtime**: deserialization is driven by JsonMapper
reading `@required`, `@maps {jsonName}` (the wire name), `@factory` (the enum or date codec) and
`@mapsBy` (the union template). This is why `opcache.save_comments` must stay enabled — see
**php-getting-started**.

## Builder class shape

`{Model}Builder` in `PaypalServerSdkLib\Models\Builders`: `init(...)` takes the required fields, one
method per optional field, `unset{Field}()` for optional-and-nullable fields, `additionalProperty(string
$name, $value)` where supported, and `build()` returning a clone. Enums get **no** builder.

## Enum class shape
```php
class {EnumClass}
{
    public const {CONSTANT} = 'value';
    // …

    private const _ALL_VALUES = [self::{CONSTANT}, …];

    /**
     * @param array|stdClass|null|string $value
     * @return array|null|string
     * @throws Exception
     */
    public static function checkValue($value) { … }
}
```

- Integer enums are identical with `int` literals and `int` in the docblock.
- `checkValue` accepts a single value **or a list/map of values**, and returns the input on success.
- An **open** enum (the spec allows values outside the list) is generated **without** `checkValue` and
  `_ALL_VALUES` — just the constants. Its absence tells you the field is not validated.
- `Environment` and `Server` in `src/` are the same shape without `checkValue`.

Usage:

```php
$status = {EnumClass}::{CONSTANT};      // a plain string/int
{EnumClass}::checkValue($userInput);    // validate early; throws Exception on a bad value
```

## Union (`@mapsBy`) template grammar

The template on a union setter is the runtime's validation contract:

```
oneOf(Car,Morning,Atom)              exactly one case must match
anyOf(Atom,Orbit)                    at least one case must match
anyOf(oneOf(Orbit,Vehicle),null)     nested; ",null" is how nullability is expressed
```

Collections append a suffix to a case (arrays and maps of a case), so a template can nest several
levels deep. Read it left to right: the outermost combinator is applied last.

There is deliberately **no** generated code to construct or destructure these. Assign the variant
instance directly; test it with `instanceof` (objects) or `is_*` (scalars) on the way back.

Response-side unions appear in the controller as `->typeGroup('<template>')` on the response handler
rather than `->type({Model}::class)`.

`doc/models/containers/*.md` documents each union's cases in prose. Note that there is **no**
`src/Models/Containers/` directory — that folder exists only in the generated markdown.

## Optional-and-nullable storage

Such a field is stored as a one-key array box, which is what makes the three states representable: the
box is empty when the field is absent and holds one entry once set, possibly to `null`. `unset{Field}()`
empties it again. A plain-optional field is stored as the value itself and serialized under
`isset(...)`, which is why it can never carry an explicit `null`.

## Additional properties

On the model (generated only when the spec allows them):

| Method | Behaviour |
| --- | --- |
| `addAdditionalProperty(string $name, $value)` | stores it. In an SDK with extended additional-properties support, the model class carries a `protected $propertyNames` array and this first throws `\InvalidArgumentException` if `$name` collides with a declared property — whether the additional properties are typed or untyped (`mixed`). Look for `$propertyNames` on the model class, not for a typed `$value` parameter |
| `findAdditionalProperty(string $name)` | the value, or **`false`** when absent |

On the builder the setter is renamed to `additionalProperty(string $name, $value): self`. It delegates
straight to the model's `addAdditionalProperty()`, so it throws too, even though the builder class has no
`$propertyNames` of its own.

## Date/time — `DateTimeHelper`

All date and date-time fields are `\DateTime` in PHP. `PaypalServerSdkLib\Utils\DateTimeHelper`
(`src/Utils/DateTimeHelper.php`, generated only when the API uses dates) does the conversion. Its
`to…` functions serialize, its `from…` functions deserialize, and the `@factory` annotation on a
setter names exactly which one the field uses — that is how you identify the wire format:

| Wire format | Serialize | Deserialize |
| --- | --- | --- |
| simple date (`YYYY-MM-DD`) | `toSimpleDate` | `fromSimpleDate`, `fromSimpleDateRequired` |
| RFC 1123 | `toRfc1123DateTime` | `fromRfc1123DateTime`, `fromRfc1123DateTimeRequired` |
| RFC 3339 | `toRfc3339DateTime` | `fromRfc3339DateTime`, `fromRfc3339DateTimeRequired` |
| Unix timestamp | `toUnixTimestamp` | `fromUnixTimestamp`, `fromUnixTimestampRequired` |

Each family also has collection variants — `…Array` / `…2DArray` on the serialize side, and
`…Array` / `…Map` / `…ArrayOfMap` / `…MapOfArray` on the deserialize side. The `…Required` variants
return a non-nullable `\DateTime`.

## Files

Binary uploads use `PaypalServerSdkLib\Utils\FileWrapper` (`src/Utils/FileWrapper.php`, generated only
when an operation takes a file parameter):

```php
FileWrapper::createFromPath(string $realFilePath, ?string $mimeType = null, ?string $filename = '')
```

plus `getFilename()` and `getMimeType()`. A binary *response*, by contrast, is typed `string`.

## Notes

- Model classes and their methods are **not** `final`, so they can be subclassed or doubled in tests.
- `__toString()` produces a readable dump via `ApiHelper::stringify` — useful in logs, not a stable
  format to parse.
- `jsonSerialize()` gives you the exact payload the SDK would send: `json_encode($model)` is the
  quickest way to check what a request body will look like before making the call.
