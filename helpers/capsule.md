# Capsule Helper

The `Helpers\Data\Capsule` class is a fluent, schema-aware data container that supports immutability, type casting, computed properties, and validation.

> Use capsules for DTOs, configuration sets, or any structured data that needs consistency.

## Creation

### make / empty / fromJson

```php
$capsule = Capsule::make(['id' => 1, 'name' => 'Anchor']);
$empty = Capsule::empty();
$data = Capsule::fromJson('{"key": "value"}');
```

## Schema & Validation

The `schema` method allows you to define allowed keys, their types (casting), and custom validation rules.

```php
$capsule->schema([
    'id'      => 'int',
    'email'   => [
        'type'     => 'string',
        'validate' => fn($val) => str_contains($val, '@'),
        'default'  => 'guest@example.com'
    ],
    'is_admin' => 'bool',
    'options'  => 'json',
    'born_at'  => 'date'
]);
```

### Supported Casts

- `int`, `float`, `string`, `bool`, `array`, `json`
- `date` (automatically parses via `DateTimeHelper`)
- Any class name (instantiates the class with the value)

## Computed Properties

Computed properties are dynamic values generated on-the-fly when accessed via `get()`.

```php
$capsule->computed('full_name', function($c) {
    return $c->get('first_name') . ' ' . $c->get('last_name');
});

echo $capsule->get('full_name');
```

## Immutability & Safety

| Method        | Behavior                                                                                |
| :------------ | :-------------------------------------------------------------------------------------- |
| `immutable()` | Seals the capsule. Any attempt to modify data will throw a `RuntimeException`.          |
| `freeze()`    | Prevents adding **new** keys but allows updating existing ones. Also seals the capsule. |
| `seal(data)`  | Shortcut for creating an immutable capsule immediately.                                 |

## Cloning & Comparison

Capsule provides methods for creating modified clones of sealed or unsealed containers.

### with / without

```php
$new = $capsule->with(['role' => 'editor']);
$stripped = $capsule->without('password', 'internal_id');
```

- Returns a **new instance** with the specified changes.
- Useful for updating immutable capsules without losing original state.

### equals

```php
if ($capsule->equals($otherCapsule)) { ... }
```

## Fluent Control Flow

Inspired by collections, Capsule supports functional-style chaining.

```php
$capsule
    ->when($isAdmin, fn($c) => $c->set('role', 'root'))
    ->tap(fn($c) => Log::info('Debug collection', $c->toArray()))
    ->fill($newData);
```

- `when(bool, callback)` / `unless(bool, callback)`
- `tap(callback)` - Execute logic without breaking the chain.
- `pipe(callback)` - Pass the capsule into a callback and return its result.

## Data Operations

- `merge(array|object, $deep = true)` - Combine datasets.
- `only(array $keys)` / `except(array $keys)` - Whitelist/Blacklist data.
- `pluck(string $key)` - Get nested data as a flat array.
- `isEmpty()` / `isNotEmpty()` - Check for internal data presence.
- `sum(?string $property = null, ?Closure $callback = null)` - Calculate sums (supports nested plucking).
- `export(callable $transformer)` - Transform and return the internal data.

## Serialization

- `toArray()`: Returns raw array.
- `toJson()`: Returns JSON string.
- `cacheKey()`: Generates a unique hash of the data for caching.

## Related

- [DTO Helper](dto.md) - For typed data objects
- [Data Helper](data.md) - Manipulation of complex datasets
- [Validation Helper](validation.md) - Deep data validation
