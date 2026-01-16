# Format Helper

The `Helpers\Format` namespace provides utilities for converting data between various formats like Arrays, JSON, XML, and standardized API responses.

> Use the `format()` global helper for quick, fluent access.

```php
// Convert to JSON with pretty print
$json = format($data)->asJson(true)->get();

// Convert nested objects to array
$array = format($object)->asArray(true)->get();
```

## Basic Converters

#### asArray

```php
asArray(mixed $data, bool $recursive = false): array
```

Converts the provided data (e.g., an object) into a raw PHP array.

- **Use Case**: Preparing an object's data for insertion into a database using standard PDO methods.
- **Example**: `format($userObject)->asArray(true)->get()`.

#### asJson

```php
asJson(array $data, bool $debug = false): string
```

Encodes an array into a JSON string. If `$debug` is true, the `JSON_PRETTY_PRINT` flag is automatically applied for readability.

- **Use Case**: Logging data to a file or returning a quick JSON body for an API error.

#### asXml

`asXml(array $data): string`

Converts an array to an XML string. Root element is `<response>`.

#### asString

Converts the input to a string. Arrays and objects are formatted as `key: value;` pairs separated by newlines.

#### asObject

`asObject(mixed $data): object`

Converts the provided data into a PHP `stdClass` object.

## API Response Formatters

#### asSuccessfulApiResponse

```php
asSuccessfulApiResponse(string $message, mixed $data = null, ?array $metadata = null): array
```

Generates a standardized success response array structure.

- **Example structure**: `['status' => true, 'message' => '...', 'data' => [...]]`.

#### asFailedApiResponse

```php
asFailedApiResponse(string|array $message, mixed $data = null): array
```

Generates a standardized failure response array structure.

- **Use Case**: Returning error messages from a validation service or an internal API in a consistent format.
- **Example**: `format()->asFailedApiResponse('User not found')`.

## FormatObject (Fluent Wrapper)

The `FormatObject` class provides a chainable interface for transformations. Each method call updates the internal data and returns the formatter instance.

#### get

`get(): mixed`

Retrieves the final transformed data from the formatter.

## Global Helper

#### format

`format(mixed $data): FormatObject`

Initializes a new `FormatObject` with the provided data.

## Related

- [Array](array-helper.md) - Deep array manipulation
- [String](string-helper.md) - Advanced string operations
- [HTTP Response](response-helper.md) - Sending formatted data to the browser
