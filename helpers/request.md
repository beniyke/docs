# Request Helper

The `Helpers\Http\Request` class provides a comprehensive interface for handling incoming HTTP requests, accessing input data, performing security checks, and inspecting the request URI.

> Use the `request()` global helper to access the current request instance.

## Input Data

#### post / get

```php
post(?string $key = null, mixed $default = null): mixed

get(?string $key = null, mixed $default = null): mixed
```

Retrieves data from `$_POST` or `$_GET`. Supports dot-notation for nested arrays.

#### all / only / exclude

```php
all(): array

only(array $keys): array

exclude(array $exclude): array
```

`all` consolidates GET, POST, and Files. `only` and `exclude` filter the consolidated input.

#### has / filled / anyIsFilled

```php
has(string|array $key): bool

filled(string|array $key): bool

anyIsFilled(array $keys): bool
```

- `has`: Key exists in input.
- `filled`: Key exists and is not empty.
- `anyIsFilled`: True if _any_ of the specified keys have values.

## Files

#### file / hasFile

```php
file(?string $key = null): mixed

hasFile(): bool
```

`file` returns a `FileHandler` instance (see [HTTP Helper](http-helper.md)) or an array of handlers for multiple uploads.

## URI & URL Information

#### path / uri

```php
path(): string

uri(): string
```

Returns the current request path/URI (e.g., `admin/users`). `uri()` is an alias for `path()`.

#### referer / callback

```php
referer(): ?string
callback(): string
```

- `referer`: Returns the `Referer` header value.
- `callback`: Returns a fully qualified URL for the current or specified callback route.

#### baseUrl / fullRoute

```php
baseUrl(?string $path = null): string
fullRoute(?string $uri = null, bool $re_route = false, array $params = []): string
```

`baseUrl` returns the site base URL + path.

`fullRoute` returns the absolute URL including query parameters.

#### host / domain / scheme

```php
host(): string
domain(): string
scheme(bool $suffix = false): string
```

- `host`: Returns the current host (e.g., `example.com`).
- `domain`: Returns the `SERVER_NAME`.
- `scheme`: Returns the protocol (e.g., `https`). If `$suffix` is true, returns `https://`.

## HTTP Methods & State

- `method(): string` - Returns the HTTP method (e.g., `POST`).
- `isPost()`, `isGet()`, `isPut()`, `isPatch()`, `isDelete()`, `isOptions(): bool`
- `isStateChanging(): bool` (POST, PUT, PATCH, DELETE)
- `isAjax(): bool`
- `isSecure(): bool`

#### cookies

```php
cookies(?string $key = null): mixed
```

Retrieves a cookie value or all cookies if no key is provided.

## Headers & Negotiation

- `header(?string $key = null): mixed`
- `bearerToken(): ?string`
- `wantsJson(): bool`
- `expectsJson(): bool` (Alias for `wantsJson`)
- `contentTypeIs(string $type): bool` - Check if `Content-Type` matches a specific type.
- `accepts(string|array $contentTypes): bool`
- `userAgent(): string`
- `ip(): string`

## Security

- `isSecurityValid(): bool` - Runs CSRF, Honeypot, and Bot detection.
- `getCsrfToken(): string` - Retrieves or generates the session CSRF token.
- `sanitize(bool $status): self` - Opt-in/out of automatic input sanitization.
- `isBot(): bool` - Checks if the requester is a known search engine bot.

## Authentication

- `user(): mixed` - Returns the currently authenticated user object.
- `token(): ?string` - Returns the current authentication token.

## PSR-7 Compliance

The `Request` class implements the PSR-7 `ServerRequestInterface`, providing methods like `getProtocolVersion()`, `getHeaders()`, `getBody()`, `getQueryParams()`, etc.

## Related

- [Response](response-helper.md) - Outgoing server responses
- [Redirect](redirect-helper.md) - Handled via Request helper
- [URL](url-helper.md) - URL generation
- [UserAgent](agent-helper.md) - Detailed agent parsing
- [Flash](flash-helper.md) - Session messages
