# DTO Helper

The `Helpers\DTO` class provides a base for creating strict, type-safe Data Transfer Objects with automatic property mapping and validation.

```php
use Helpers\DTO;

class CreateUserDTO extends DTO
{
    public string $name;
    public string $email;
    public ?int $age = null;
    public bool $active = true;
}

$user = new CreateUserDTO([
    'name' => 'John Doe',
    'email' => 'john@example.com'
]);

echo $user->name; // John Doe
```

## Defining DTOs

DTOs are defined by extending the `Helpers\DTO` class and defining public properties with appropriate type hints.

- **Required Properties**: Properties without a default value or nullability are considered required.
- **Optional Properties**: Can be marked as nullable (`?int`) or assigned a default value.
- **Readonly Support**: Works with PHP `readonly` properties.

## Class Methods

#### toArray

```php
toArray(): array
```

Converts the DTO properties into a raw associative array.

- **Use Case**: Passing data from a DTO into a view or a database query builder.

#### getData

```php
getData(): Data
```

Returns a `Helpers\Data` instance wrapping the DTO's properties.

- **Use Case**: Using the DTO as a source for fluent data manipulation.

#### isValid

```php
isValid(): bool
```

Returns `true` if all properties designated as "required" (those without default values) have been provided.

#### getErrors

```php
getErrors(): array
```

Returns an array of descriptive error messages if any required properties are missing.

- **Example**: `["The required property 'email' is missing."]`.

### Handling Missing Data

Unlike simple objects, the `DTO` base class tracks missing required fields during construction.

```php
$dto = new CreateUserDTO(['name' => 'John']);

if (!$dto->isValid()) {
    foreach ($dto->getErrors() as $error) {
        echo $error; // "The required property 'email' is missing."
    }
}
```

## Related

- [Data](data-helper.md) - ArrayAccess wrapper
- [Capsule](capsule-helper.md) - Immutable data container
- [Validation](validation-helper.md) - Input validation schemas
