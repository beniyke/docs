# HTTP Helpers

The `Helpers\Http` namespace provides a comprehensive suite of tools for managing HTTP requests, server responses, sessions, cookies, and client-side communications.

## Main HTTP Components

For detailed documentation on the primary HTTP systems, refer to:

| Component     | Description                                  | Documentation                  |
| :------------ | :------------------------------------------- | :----------------------------- |
| **Request**   | Incoming request data, headers, and security | [request](request-helper.md)   |
| **Response**  | Outgoing server responses, JSON, and files   | [response](response-helper.md) |
| **Curl**      | Fluent HTTP client for external requests     | [curl](curl-helper.md)         |
| **Session**   | persistent state management                  | [session](session-helper.md)   |
| **Flash**     | One-time session messages and old input      | [flash](flash-helper.md)       |
| **UserAgent** | Browser, OS, and device detection            | [agent](agent-helper.md)       |

## HTTP Utilities

### FileHandler

The `FileHandler` class provides a wrapper around `$_FILES` for safe and easy upload management.

#### moving files

```php
move(string $target_path, ?string $name = null): bool

moveSecurely(string $destination, array $options = []): bool
```

`move` is the basic upload handler. `moveSecurely` integrates with `FileUploadValidator` to ensure the file meets specific criteria before being stored.

#### validation

```php
validate(array $options): bool

validateWith(FileUploadValidator $validator): bool

getValidationError(): ?string
```

Supports validation of size, extension, and MIME type.

#### metadata & status

| Method                    | Description                                                 |
| :------------------------ | :---------------------------------------------------------- |
| `isValid()`               | Returns true if the upload was successful and has no errors |
| `isEmpty()`               | Returns true if no file was uploaded in this slot           |
| `hasError()`              | Returns true if PHP encountered an upload error             |
| `getErrorMessage()`       | Returns a human-readable PHP upload error message           |
| `getClientSize()`         | Returns the file size in bytes as reported by the client    |
| `getClientOriginalName()` | Returns the original filename                               |
| `getExtension()`          | Guesses the correct extension based on the MIME type        |
| `getMaxFilesize()`        | Returns the maximum allowed upload size from `php.ini`      |

### Header

A convenient interface for manipulating HTTP header collections.

```php
header()->set('X-Content-Type-Options', 'nosniff');

header()->get('Content-Type');

header()->has('Date');

header()->remove('X-Powered-By');

header()->all(); // Returns all headers
```

### Cookie

Simplified cookie management with strong security defaults.

#### set

```php
set(string $name, string $value = '', int|DateTimeInterface $expiry = 0, ...): bool
```

Sets a cookie with defaults: `Secure: true`, `HttpOnly: true`, `SameSite: Lax`.

#### configureSessionCookie

```php
static configureSessionCookie(int $lifetime, string $path = '/', ?string $domain = null, bool $secure = true, bool $httpOnly = true, string $sameSite = 'Lax'): void
```

Global configuration for session cookies.

- **Expiry**: Can be an integer (seconds from now) or a `DateTime` instance.
- **Example**: `cookie()->set('remember_me', $token, 3600 * 24 * 30);`

#### get / has / forget

```php
cookie()->has('session_id');

cookie()->forget('user_prefs'); // Expires the cookie immediately
cookie()->delete('user_prefs'); // Alias for forget()
```

### Server

Extends the `Header` class to extract and normalize environment data from `$_SERVER`.

```php
$headers = server()->getHeaders();
```

Automatically resolves authorization headers, including **Basic Auth** (decoding it into `PHP_AUTH_USER`/`PW`) and **Bearer Tokens**.

## Related

- [URL](url-helper.md) - URL generation and route assistance
- [Redirect](redirect-helper.md) - Specialized redirect responses
- [Route](route-helper.md) - Route information and parameters
- [Request URI](request-uri-helper.md) - URI parsing and segments
