# Cache Helper

The `Helpers\File\Cache` class implements a robust file-based caching system with support for TTL, tagging, locking, stale-while-revalidate patterns, and LRU eviction.

> Use the `cache()` global function for convenient access.

```php
// Simple caching
$value = cache()->remember('user_count', 300, fn() => User::count());

// Read from cache
$data = cache()->read('api_response');

// Write to cache with TTL
cache()->write('settings', $config, 3600);
```

## Factory Methods

### create

```php
static create(string $path = 'cache', string $prefix = '', string $extension = 'cache'): self
```

Creates a cache instance. The path is relative to `App/storage`.

```php
$cache = Cache::create('api_cache', 'v1_', 'json');
```

### withPath

```php
withPath(string $subPath): self
```

Returns a new instance scoped to a subdirectory. Useful for organizing cache by feature.

```php
$userCache = cache()->withPath('users');
$productCache = cache()->withPath('products');

$userCache->write('profile_123', $userData, 600);
```

## Core Operations

### write

```php
write(string $key, mixed $value, int $ttl = 0): bool
```

Stores data in the cache.

| Parameter | Description                                            |
| :-------- | :----------------------------------------------------- |
| `$key`    | Unique identifier for the cached item                  |
| `$value`  | Data to store (automatically serialized)               |
| `$ttl`    | Time-to-live in seconds. Use `0` for permanent storage |

```php
// Cache for 1 hour
cache()->write('api_response', $data, 3600);

// Cache permanently
cache()->write('static_config', $config, 0);
```

### read

```php
read(string $key, mixed $default = null): mixed
```

Retrieves data from the cache. Returns `$default` if the item is missing or expired.

```php
$user = cache()->read('user_123');

// With default value
$settings = cache()->read('app_settings', []);
```

### has

```php
has(string $key): bool
```

Checks if a valid (non-expired) cache entry exists.

```php
if (cache()->has('expensive_calculation')) {
    $result = cache()->read('expensive_calculation');
} else {
    $result = performExpensiveCalculation();
    cache()->write('expensive_calculation', $result, 3600);
}
```

### delete

```php
delete(string $key): bool
```

Removes a specific cache entry.

```php
cache()->delete('user_123');
```

### clear

```php
clear(): bool
```

Removes all items from the current cache directory.

```php
cache()->clear();

// Clear a specific namespace
cache()->withPath('api_responses')->clear();
```

### keys

```php
keys(): array
```

Returns all keys stored in the current cache directory.

```php
$allKeys = cache()->keys();
// ['user_123', 'settings', 'api_response']
```

## Remember Patterns

### remember

```php
remember(string $key, int $seconds, callable $callback): mixed
```

The most common caching pattern: read from cache if available; otherwise, execute the callback, store the result, and return it.

```php
// Cache database query for 5 minutes
$users = cache()->remember('active_users', 300, function () {
    return User::where('active', true)->get();
});

// Cache API response for 1 hour
$weather = cache()->remember('weather_data', 3600, fn() =>
    WeatherApi::getCurrentConditions()
);
```

### permanent

```php
permanent(string $key, callable $callback): mixed
```

Like `remember()`, but stores the result indefinitely (TTL = 0).

```php
$config = cache()->permanent('app_config', fn() =>
    Config::loadFromDatabase()
);
```

### rememberWithStale

```php
rememberWithStale(string $key, int $ttl, callable $callback): mixed
```

Implements the "stale-while-revalidate" pattern. Returns stale data immediately while refreshing in the background.

- Prevents cache stampedes on high-traffic endpoints
- Uses locking to ensure only one process refreshes

```php
$products = cache()->rememberWithStale('product_list', 300, function () {
    return Product::with('categories')->get();
});
```

## Tagging

Tags allow you to group related cache items for bulk invalidation.

### tags

```php
tags(array $tags): self
```

Sets tags for the next write operation.

```php
cache()
    ->tags(['users', 'api'])
    ->write('user_123', $userData, 3600);

cache()
    ->tags(['products', 'api'])
    ->write('product_list', $products, 1800);
```

### flushTags

```php
flushTags(array $tags): bool
```

Invalidates all cache items associated with any of the given tags.

```php
// Invalidate all user-related cache
cache()->flushTags(['users']);

// Invalidate multiple tags
cache()->flushTags(['users', 'products']);
```

## Locking

Prevents cache stampedes when multiple requests try to regenerate expired cache simultaneously.

### acquireLock

```php
acquireLock(string $key, int $timeout = 10): bool
```

Attempts to acquire an exclusive lock. Returns `true` if successful, `false` if another process holds the lock.

### releaseLock

```php
releaseLock(string $key): bool
```

Releases a previously acquired lock.

```php
$key = 'expensive_report';

if (cache()->acquireLock($key, 30)) {
    try {
        $data = generateExpensiveReport();
        cache()->write($key, $data, 3600);
    } finally {
        cache()->releaseLock($key);
    }
} else {
    // Another process is generating, wait or return stale
    sleep(1);
    $data = cache()->read($key);
}
```

## LRU Eviction

The cache supports Least Recently Used (LRU) eviction to prevent unbounded growth.

### setMaxItems

```php
setMaxItems(int $maxItems): self
```

Sets the maximum number of items before LRU eviction kicks in.

```php
cache()->setMaxItems(1000);
```

### enforceLimit

```php
enforceLimit(): void
```

Manually triggers LRU eviction if the cache exceeds the max items limit.

## Metrics

### getMetrics

```php
getMetrics(): array
```

Returns cache performance statistics.

```php
$metrics = cache()->getMetrics();
// [
//     'hits' => 1523,
//     'misses' => 47,
//     'writes' => 89,
//     'hit_rate' => 0.97,
//     'total_size' => 2457600
// ]
```

### getCacheSize

```php
getCacheSize(): int
```

Returns the total size of cached files in bytes.

### resetMetrics

```php
resetMetrics(): void
```

Resets the hit/miss/write counters.

## Advanced Features

### addJitter

```php
addJitter(int $ttl): int
```

Adds random variance to TTL values to prevent "thundering herd" problems where many cache entries expire simultaneously.

### writeWithExpiry

```php
writeWithExpiry(string $key, mixed $value, int $ttl): bool
```

Writes with automatic TTL jitter applied.

## Configuration

The cache system reads configuration from `App/Config/cache.php`:

```php
return [
    'default_ttl' => 3600,
    'max_items' => 10000,
    'jitter_percentage' => 10,
];
```

## Examples

### Caching API Responses

```php
$response = cache()->remember('github_repos', 1800, function () {
    return curl()
        ->withToken(config('github.token'))
        ->get('https://api.github.com/user/repos')
        ->send()
        ->json();
});
```

### User-Specific Caching

```php
$userCache = cache()->withPath("users/{$userId}");

$preferences = $userCache->remember('preferences', 3600, fn() =>
    UserPreference::where('user_id', $userId)->first()
);
```

### Invalidating Related Cache

```php
// When a product is updated
function updateProduct(Product $product, array $data): void
{
    $product->update($data);

    // Invalidate all product-related cache
    cache()->flushTags(['products', "product_{$product->id}"]);
}
```

## Related

- [FileSystem](filesystem-helper.md) - Low-level file operations
- [Paths](paths-helper.md) - Application path helpers
