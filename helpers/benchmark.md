# Benchmark Helper

The `Helpers\Benchmark` class provides utilities for performance profiling, allowing you to measure execution time and memory usage for specific operations or code blocks.

> [!TIP]
> Use the `benchmark()` global helper for quick access.

## Basic Usage

#### start / stop

```php
static start(string $key): void
static stop(string $key): float
```

`start` begins recording for a given key. `stop` ends the timer and returns the duration in **milliseconds**.

```php
Benchmark::start('db_query');
// ... run query ...
$time = Benchmark::stop('db_query');

echo "Query took {$time}ms";
```

#### measure

```php
static measure(string $key, callable $callback): mixed
```

Wraps a callback in a benchmark timer. It returns the result of the callback itself.

```php
$users = Benchmark::measure('fetch_users', function() {
    return User::all();
});
// 'fetch_users' timer is now recorded and stopped
```

## Retrieval

#### get / memory

```php
static get(string $key): ?float
static memory(string $key): ?int
```

- `get`: Retrieves the duration (ms) of a _previously stopped_ timer.
- `memory`: Retrieves the memory difference (bytes) used during the timer's execution.

#### getAll

```php
static getAll(): array
```

Returns an associative array of all recorded timers and their metrics.

```php
$stats = Benchmark::getAll();
/*
[
    'fetch_users' => [
        'time' => 12.5,  // ms
        'memory' => 1024 // bytes
    ]
]
*/
```

## Maintenance

#### reset

```php
static reset(): void
```

Clears all recorded timers from memory.

## Related

- [Data Helper](data-helper.md) - For analyzing sets of benchmark data
- [VarDump Helper](vardump-helper.md) - Displaying performance stats during development
