# Array Helper

The Array helper provides a powerful set of tools for manipulating arrays and collections. It consists of the static `ArrayCollection` utility and the fluent `Collections` wrapper.

> Use the `arr()` global function for convenient access.

- `arr($data)` returns a fluent `Collections` instance.
- `arr()` returns an `ArrayCollection` instance.

> See [Functions Reference](functions.md#arr) for details.

```php
// Using global helper fluently
$result = arr($users)
    ->where('active', true)
    ->pluck('name')
    ->get();

// Using static class directly
use Helpers\Array\ArrayCollection;
$average = ArrayCollection::avg($prices);
```

## ArrayCollection

`Helpers\Array\ArrayCollection` provides static methods for advanced array manipulation, supporting dot-notation, deep mapping, and statistical calculations.

### Access & Modification

#### set

```php
set(array $array, string $key, mixed $value): array
```

Sets a value within an array using dot notation for nested structures.

- **Use Case**: Dynamically building configuration or state objects.
- **Example**: `ArrayCollection::set($data, 'app.ui.theme', 'dark')`.

#### value

```php
value(array $array, string $key, mixed $default = null): mixed
```

Retrieves a value from an array using dot notation. Returns `$default` if the key is not found.

- **Use Case**: Safely accessing nested data from parsed JSON or configuration files.
- **Example**: `arr($settings)->value('mail.smtp.host', 'localhost')`.

#### getOrSet

```php
getOrSet(array &$array, string $key, mixed $default): mixed
```

Retrieves a value by key. If the key doesn't exist, it sets the value to `$default` and returns it.

- **Use Case**: Initializing a nested configuration array or cache bucket only if it's missing.

#### hasAll

```php
hasAll(array $array, array $keys): bool
```

Checks if all of the provided dot-notation keys exist in the array and are not null.

- **Example**: `ArrayCollection::hasAll($data, ['user.name', 'user.email'])`.

#### has / exists

```php
has(array $array, string $key): bool

exists(array $array, string $key): bool
```

`has` returns true if the key exists AND the value is not null (dot notation supported).

`exists` returns true even if the value is null.

- **Use Case**: Distinguishing between a key with a `null` value and a key that is entirely absent.

#### forget

```php
forget(array $array, string|array $keys): array
```

Recursively removes one or more keys using dot notation.

- **Use Case**: Stripping authentication tokens or internal IDs before sending an array to an API.
- **Example**: `arr($user)->forget(['password', 'profile.token'])`.

#### only / exclude

```php
only(mixed $data, array $filter): array|object

exclude(mixed $data, array $exclude): array|object
```

Whitelist or blacklist keys from an array or object.

- **Use Case**: Sanitizing inputs for a database update to prevent Mass Assignment vulnerabilities.

### Filtering & Extraction

#### where

```php
where(array $array, string $field, mixed $value, bool $strict = true): array
```

Filters a list of arrays or objects where a specific field matches a value.

- **Use Case**: Filtering a list of products by category or users by status.
- **Example**: `arr($products)->where('category', 'electronics')`.

#### whereIn / whereNotIn

```php
whereIn(array $array, string $field, array $collection, bool $strict = true): array

whereNotIn(array $array, string $field, array $collection, bool $strict = true): array
```

Filters items based on whether a field's value exists within a provided list.

- **Use Case**: Filtering posts based on a list of selected tag IDs.

#### pluck

```php
pluck(array $array_list, string $fieldName = 'id'): array
```

Exatracts a single field from every item in the list into a flat array.

- **Use Case**: Getting an array of emails from a list of user objects for a mailing list.
- **Example**: `arr($users)->pluck('email')`.

#### partition

`partition(array $array, callable $callback): array`

Divides the array into two arrays: `[passed, failed]`.

```php
[$active, $inactive] = ArrayCollection::partition($users, fn($u) => $u['active']);
```

#### unique

`unique(array $array, bool $keep_keys = false): array`

Removes duplicate values.

#### uniqueBy

`uniqueBy(array $data, string $target_key): array`

Removes duplicate items based on a specific key's value.

### Transformation

#### map

```php
map(array $array, callable $function): array
```

Applies a callback to every value in the array.

- **Use Case**: Normalizing a list of strings (e.g., lowercase) or calculating taxes for a list of prices.

#### mapDeep

```php
mapDeep(array $array, callable $callback, bool $on_no_scalar = false): array
```

Recursively applies a callback function to every element in the array.

- `$on_no_scalar`: If true, applies the callback to non-scalar values (like objects) as well.

#### mapWithKeys

```php
mapWithKeys(array $array, callable $callback): array
```

Iterates through the array and passes each value to the given callback. The callback should return an associative array containing a single key/value pair.

- **Use Case**: Re-keying a list based on a unique attribute.
- **Example**: `arr($users)->mapWithKeys(fn($u) => [$u['email'] => $u['name']])`.

#### reduce

`reduce(array $array, callable $callback, mixed $initial = null): mixed`

Reduces the array to a single value using a callback.

#### flatten

`flatten(array $array, bool $preserve_keys = true): array`

Flattens a multi-dimensional array.

#### build

`build(array $array, string $keyField, string|array|null $valueField = null): array`

Builds an associative array keyed by a specific field.

#### indexByCompoundKey

`indexByCompoundKey(array $array, array $keyFields, ?string $valueField = null, string $separator = '-', bool $asList = false): array`

Indexes an array using a composite key made of multiple fields.

- `$keyFields`: Array of fields to combine for the key.
- `$valueField`: Optional. Specific field to use as value.
- `$separator`: Glue for the compound key (default '-').
- `$asList`: If true, returns an array of items for each compound key (grouping).

#### replaceKeys

`replaceKeys(array $array, array $replace, bool $multiple = true): array|object`

Replaces keys with new names.

#### replaceValues

`replaceValues(array $array, array $replacements): array`

Replaces values based on a map of rules or closures.

### Statistics & Math

#### avg / mean

```php
avg(array $array, ?string $field = null): float
mean(array $array, ?string $field = null): float
```

Calculates the numerical average of values. `mean` is an alias for `avg`.

#### median

```php
median(array $array, ?string $field = null): float
```

Calculates the median (middle value) of an array of numbers.

#### mode

```php
mode(array $array, ?string $field = null): array
```

Calculates the mode (most frequently occurring value). Returns an array as there can be multiple modes.

#### variance / stdDev

```php
variance(array $array, ?string $field = null): float
stdDev(array $array, ?string $field = null): float
```

Calculates statistical variance and standard deviation.

#### sum

```php
sum(array $array): float|int
```

Returns the sum of all numerical values.

#### max / min

```php
max(array $array): mixed
min(array $array): mixed
```

Returns the maximum or minimum value.

### Sorting & Ordering

#### sortByField

```php
sortByField(array $records, string $field, bool $reverse = false): array
```

Sorts an array of associative arrays or objects based on the value of a specific field.

- **Use Case**: Sorting users by their registration date or products by price.
- **Example**: `arr($users)->sortByField('created_at', true)` (descending).

#### orderKeys

```php
orderKeys(array $array, array $order): array
```

Reorders an associative array's keys according to a predefined list.

- **Use Case**: Ensuring a specific field order when exporting data to CSV or displaying a table.

#### reverse

`reverse(array $array): array`

Reverses the array order.

### Iteration & Checks

#### each

`each(array $array, callable $callback): array`

Iterates over items. Return `false` to break the loop.

#### contains

`contains(array $array, mixed $value, bool $return_key = false): mixed`

Checks if a value (or list of values) exists in the array.

- If `$value` is an array of items, it returns `true` if **any** of them are found.
- If `$return_key` is true, it returns the key(s) instead of a boolean.

### Utilities & Conversion

#### first / last

```php
first(array $array): mixed

last(array $array): mixed
```

Retrieves the first or last element of an array without shifting the internal pointer.

- **Example**: `arr($news)->first()`.

#### firstKey / lastKey

```php
firstKey(array $array): mixed

lastKey(array $array): mixed
```

Retrieves the first or last key of an array.

#### remove

```php
remove(array $array, string $key): array
```

Removes an item from the array by key (non-destructive, does not support dot notation).

#### push / prepend

```php
push(array $array, mixed $item): array
prepend(array $array, mixed $item): array
```

Pushes an item (or array of items) onto the end or beginning of the array.

#### combine

`combine(array $keys, array $values): array`

Combines keys and values arrays.

#### zip

`zip(array $array, array $other): array`

Zips two arrays together into pairs.

#### flip

`flip(array $array): array`

Exchanges keys with values.

#### getKeys

`getKeys(array $array): array` / `values(array $array): array`

Returns array keys or values.

#### groupByKey

`groupByKey(array $array_list, string $key = 'id'): array`

Groups an array of arrays/objects by a specified key.

#### prependKeyed

`prependKeyed(array $array, string $key, mixed $value): array`

Adds an element to the beginning while preserving keys.

#### attach

`attach(array $array, array $item): array`

Merges two arrays non-destructively, prioritizing existing keys.

#### rebase

`rebase(array $array): array`

Re-indexes the array numerically from 0.

#### chunk

`chunk(array $array, int $size, bool $preserveKeys = false): array`

Splits an array into smaller chunks.

#### take

`take(array $array, int $limit): array`

Takes a specific number of elements from the start (positive) or end (negative).

#### limit

`limit(array $array, int $limit, int $offset = 0): array`

Returns a slice of the array.

#### random

`random(array $array, int $count = 1): mixed`

Retrieves random elements.

#### shuffle

`shuffle(array $array): array`

Shuffles array items randomly.

#### shift

`shift(array $array): mixed` / `pop(array $array): mixed`

Removes and returns the first/last item.

#### count

`count(array $array, bool $recursive = false): int`

Counts elements.

#### isEmpty

`isEmpty(array $array): bool`

Checks if the array is empty.

#### isAssoc

`isAssoc(array $array): bool`

Checks if the array is associative.

#### isArrayOfArrays

`isArrayOfArrays(array $array): bool`

Checks if all elements are arrays.

#### isMultiDimensional

`isMultiDimensional(mixed $data): bool`

Checks if the data contains nested structures.

#### isEqual

`isEqual(array $array1, array $array2): bool`

Checks if two arrays contain the same elements (order independent).

#### cleanDeep

`cleanDeep(array $array): array`

Recursively removes empty strings, nulls, and empty arrays.

#### clean

`clean(array $haystack, ?callable $callback = null): array`

Filters an array using a custom callback.

#### toAssoc

`toAssoc(array $list, ?string $keyField = null, ?string $valueField = null): array`

Converts a list to an associative array.

#### wrap

`wrap(mixed $object): array`

Wraps a value in an array (returns empty array for null).

#### toComment

`toComment(array $data, int $level = 0): string`

Converts an array to a pretty-printed comment string.

#### safeImplode

`safeImplode(mixed $body, string $glue = ' '): string`

Safely implodes mixed values (handles nested structures).

## Collections

`Helpers\Array\Collections` is a fluent wrapper around `ArrayCollection`.
All static methods from `ArrayCollection` are available as instance methods.

### How it works

- If a method returns an **array**, the collection updates its internal state and returns `$this` (Fluent).
- If a method returns a **scalar** (e.g., `avg()`, `count()`, `first()`), it returns the value directly.

### Core Methods

#### make

`make(mixed $array = []): self`

Creates a new collection instance.

#### get

`get(): mixed`

Returns the underlying array.

#### all

`all(): FormatObject`

Returns the data as a `FormatObject` for easy object-access.

#### put

`put(string $key, mixed $value): self`

Sets a value directly at the specified key.

#### reset

`reset(array $items = []): self`

Replaces the collection items.

#### custom

`custom(callable $function): self`

Applies a custom transformation to the collection.

#### count

`count(): int`

Returns the number of items.

#### Example

```php
$collection = arr($data)
    ->put('new_key', 'value')
    ->forget('old_key')
    ->sortByField('name')
    ->values(); // Returns fluent self with re-indexed items

$average = $collection->avg('price'); // Returns float directly
```

## Related

- [Data Helper](data-helper.md) - Immutable data wrapper
- [Format Helper](format-helper.md) - Converting arrays to other formats
- [Number Helper](number-helper.md) - Mathematical operations
