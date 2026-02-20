# Data Helper

The `Helpers\Data\Data` class is a wrapper around the `Collections` class, providing an `ArrayAccess` and magic property interface. It is useful for managing structured datasets with dot-notation support.

```php
use Helpers\Data\Data;

$data = Data::make(['user' => ['id' => 1]]);

echo $data->get('user.id'); // 1
echo $data['user.id'];      // 1 (ArrayAccess)
echo $data->user;           // ['id' => 1] (Magic Prop)
```

## Creation

### make

```php
static make(array $data, ?array $only = null): self
```

Initializes a new Data wrapper. If the `$only` array is provided, the data will be filtered to include only those keys before initialization.

- **Use Case**: Quickly wrapping a database result or API response for easier property access.

## Retrieval

### get

```php
get(string|int $key, mixed $default = null): mixed
```

Retrieves a value from the data set. Supports dot-notation for nested structures.

- **Example**: `$data->get('profile.notifications.email', true)`.

### data

```php
data(): array
```

Returns the entire dataset as a raw associative array.

- **Note**: This method automatically converts empty strings (`''`) to `null`.
- **Use Case**: Exporting the data back to a format suitable for database insertion or JSON encoding.

## Checks

### has

```php
has(string|array $keys): bool
```

Checks if a key (or an array of keys) exists in the data and is not null.

### filled

```php
filled(array $keys): bool
```

Determines if all specified keys are present and contain "truthy" data (not empty strings, not null).

- **Use Case**: Validating that required fields from a form submission actually contain data before processing.

## Transformation

> All transformation methods return a **new instance**.

### add

`add(array $items): self`

Appends new items.

### remove

`remove(array $keys): self`

Removes items by keys.

### update

`update(array $items): self`

Updates values using a key-value map.

### select

`select(array $keys): self`

Returns a new instance with only the selected keys.

## Related

| Component       | Description                           | Documentation                      |
| :-------------- | :------------------------------------ | :--------------------------------- |
| **Capsule**     | Immutable data container with schema  | [capsule](capsule.md)              |
| **DTO**         | Data Transfer Objects with validation | [dto](dto.md)                      |
| **Collections** | Fluent array manipulation             | [array](array.md)                  |
| **Routine**     | Sequential tasks and data pipelining  | [routine](routine.md)              |
| **Request**     | Handle incoming request data          | [request](request.md)              |
| **Validation**  | Data validation and rule engine       | [validation](validation.md)     |
