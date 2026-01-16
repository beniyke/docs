# Session Helper

The `Helpers\Http\Session` class provides a secure and fluent interface for managing PHP sessions, incorporating security best practices like periodic ID regeneration.

> Use the `session()` global helper to access the session instance or get/set values.

## Basic Operations

#### get

```php
get(string $key, mixed $default = null): mixed
```

Retrieves an item from the session. Returns `$default` if the key doesn't exist.

- **Use Case**: Fetching the authenticated user's ID or preferred theme setting.
- **Example**: `session()->get('user_id')`.

#### set

```php
set(string $key, mixed $value): void
```

Stores an item in the session.

- **Use Case**: Storing a unique search query or tracking a multi-step form progress.
- **Example**: `session()->set('search_query', 'anchor framework')`.

#### has

```php
has(string $key): bool
```

Determines if a key is present in the session and is not null.

- **Use Case**: Checking if a user has already seen a specific announcement.

#### delete

```php
delete(string|array $keys): void
```

Removes one or more items from the session.

- **Use Case**: Clearing a specific cached flag or removing a shopping cart item.
- **Example**: `session()->delete(['cart_id', 'coupon_code'])`.

#### all

```php
all(): array
```

Returns all stored session data as an associative array.

- **Use Case**: Debugging session state or exporting session data.

#### getId

```php
getId(): string
```

Returns the current session ID.

## Batch Operations

#### setMultiple

```php
setMultiple(string $identity, array $data): void
```

Merges an array of data into a specific sub-key in the session.

- **Use Case**: Grouping related settings under a single top-level key.
- **Example**: `session()->setMultiple('settings', ['theme' => 'dark', 'lang' => 'en'])`.

#### flush

```php
flush(): void
```

Clears every item from the current session.

- **Use Case**: Logging out a user or resetting application state.

#### clearAllExcept

```php
clearAllExcept(array $excludedKeys = []): void
```

Wipes the session but preserves specific keys.

- **Use Case**: Clearing user data during logout but keeping the "last_visited_url" for redirection.

## Security

#### regenerateId

```php
regenerateId(): void
```

Forces a new session ID for the current session.

- **Use Case**: Preventing **Session Fixation** attacks by regenerating the ID immediately after a user logs in.

#### periodicRegenerate

```php
periodicRegenerate(): void
```

Automatically regenerates the session ID at defined intervals (e.g., every 5 minutes).

- **Use Case**: Reducing the window of opportunity for an attacker to use a hijacked session ID.

#### destroy

```php
destroy(): void
```

Terminates the session completely, clearing both server-side data and the client-side cookie.

- **Use Case**: Ultimate logout or security breach mitigation.

## Related

- [Flash](flash-helper.md) - Temporary session messages
- [Cookie](cookie-helper.md) - Low-level cookie management
- [Request](request-helper.md) - Access session via request object
