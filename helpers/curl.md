# HTTP Client (Curl)

The `Helpers\Http\Client\Curl` class is a powerful, fluent HTTP client for making external requests. It supports concurrency, asynchronous operations, automatic retries, and batch processing with lifecycle hooks.

> Use the `curl()` global function for convenient access.

```php
// Simple GET request
$response = curl()->get('https://api.example.com/users')->send();

if ($response->successful()) {
    $users = $response->json();
}
```

## Configuration

### headers

```php
headers(array $headers): self
```

Sets multiple HTTP headers for the request.

```php
curl()->headers([
    'X-App-ID' => '12345',
    'Accept-Language' => 'en-US'
]);
```

### withHeader

```php
withHeader(string $name, string $value): self
```

Sets a single HTTP header.

```php
curl()->withHeader('X-Custom-Header', 'value');
```

### timeout

```php
timeout(int $milliseconds): self
```

Sets the maximum time to wait for the request to complete. Default is `30000ms`.

```php
curl()->timeout(5000)->get($url)->send(); // 5 second timeout
```

### retry

```php
retry(int $times, int $delayMs = 100): self
```

Configures automatic retry on failed requests.

- **Use Case**: Handling transient network issues or flaky third-party services.

```php
curl()->retry(3, 500)->get($url)->send(); // 3 attempts, 500ms delay
```

### withoutSslVerification

```php
withoutSslVerification(): self
```

Disables SSL peer and host verification.

> Only use in development/local environments. Never disable in production.

```php
curl()
    ->withoutSslVerification()
    ->get('https://self-signed.local')
    ->send();
```

## Authentication

### withToken

```php
withToken(string $token, string $type = 'Bearer'): self
```

Sets the `Authorization` header with a bearer token.

```php
curl()
    ->withToken('my-api-key')
    ->get($url)
    ->send();
// Authorization: Bearer my-api-key

curl()
    ->withToken('my-key', 'Token')
    ->get($url)
    ->send();
// Authorization: Token my-key
```

### withBasicAuth

```php
withBasicAuth(string $username, string $password): self
```

Configures HTTP Basic Authentication.

```php
curl()
    ->withBasicAuth('user', 'pass')
    ->get($url)
    ->send();
```

### withDigestAuth

```php
withDigestAuth(string $username, string $password): self
```

Configures HTTP Digest Authentication.

```php
curl()
    ->withDigestAuth('user', 'pass')
    ->get($url)
    ->send();
```

## Content Types

### asJson

```php
asJson(): self
```

Sets `Content-Type: application/json` and automatically JSON-encodes request data.

```php
curl()
    ->asJson()
    ->post($url, ['key' => 'value'])
    ->send();
```

### asForm

```php
asForm(): self
```

Sets `Content-Type: application/x-www-form-urlencoded`.

```php
curl()
    ->asForm()
    ->post($url, ['email' => 'user@example.com'])
    ->send();
```

### asRaw

```php
asRaw(string $data): self
```

Sends a raw string payload (XML, plain text, etc.).

```php
curl()
    ->asRaw('<xml>data</xml>')
    ->post($url, null)
    ->send();
```

### attach

```php
attach(string $name, string $path, ?string $mimeType = null, ?string $fileName = null): self
```

Attaches a file for `multipart/form-data` uploads.

```php
    curl()
    ->attach('document', '/path/to/file.pdf', 'application/pdf')
    ->post($url, ['description' => 'My file'])
    ->send();
```

## HTTP Methods

All methods return `self` for chaining. Call `send()` to execute.

| Method     | Signature                                       |
| :--------- | :---------------------------------------------- |
| **GET**    | `get(string $url): self`                        |
| **POST**   | `post(string $url, mixed $data): self`          |
| **PUT**    | `put(string $url, mixed $data): self`           |
| **PATCH**  | `patch(string $url, mixed $data): self`         |
| **DELETE** | `delete(string $url, mixed $data = null): self` |
| **SEND**   | `send(): Response`                              |

#### download

```php
download(string $url, string $destination): bool
```

Downloads a file from the specified URL directly to the destination path.

```php
// GET
$response = curl()
    ->get('https://api.com/users')
    ->send();

// POST with JSON
$response = curl()
    ->asJson()
    ->post('https://api.com/users', [
    'name' => 'John',
    'email' => 'john@example.com'
])->send();

// DELETE
$response = curl()
    ->delete('https://api.com/users/123')
    ->send();
```

## Query Parameters

### withQueryParameters

```php
withQueryParameters(array $params): self
```

Appends query string parameters to the URL.

```php
    curl()
    ->withQueryParameters(['page' => 1, 'limit' => 20])
    ->get('https://api.com/users')
    ->send();
// Requests: https://api.com/users?page=1&limit=20
```

## Concurrency & Async

### pool

```php
static pool(callable $callback): Batch
```

Creates a batch of named requests to be executed in parallel.

```php
$batch = Curl::pool(fn() => [
        'users' => curl()->get('https://api.com/users'),
        'posts' => curl()->get('https://api.com/posts'),
        'comments' => curl()->get('https://api.com/comments')
    ])
    ->send();

$users = $batch->getResults()['users']->json();
$posts = $batch->getResults()['posts']->json();
```

### concurrent

```php
static concurrent(callable $callback): array
```

Executes multiple requests concurrently and returns an array of `Response` objects.

```php
$responses = Curl::concurrent(fn() => [
    curl()->get('https://api.com/endpoint1'),
    curl()->get('https://api.com/endpoint2'),
]);

foreach ($responses as $response) {
    echo $response->httpCode();
}
```

### async

```php
async(): Promise
```

Starts an asynchronous request that can be awaited later.

```php
$promise = curl()->get($url)->async();

// ... do other work ...

$response = $promise->wait(); // Blocks until complete
```

## Batch Processing

The `Batch` class provides lifecycle hooks for advanced control over parallel requests.

### Lifecycle Hooks

| Method                        | Description                                          |
| :---------------------------- | :--------------------------------------------------- |
| `before(Closure $callback)`   | Called before batch execution starts                 |
| `progress(Closure $callback)` | Called after each individual request completes       |
| `then(Closure $callback)`     | Called after all requests complete successfully      |
| `catch(Closure $callback)`    | Called if any request fails (stops on first failure) |
| `finally(Closure $callback)`  | Always called after batch completes                  |

#### Example with Hooks

```php
$batch = Curl::pool(fn() => [
    'users' => curl()->get('https://api.com/users'),
    'posts' => curl()->get('https://api.com/posts'),
])
->before(function () {
    echo "Starting batch...\n";
})
->progress(function ($key, $response, $batch) {
    echo "Completed: $key - Status: " . $response->httpCode() . "\n";
})
->then(function ($batch) {
    echo "All requests completed successfully!\n";
})
->catch(function ($key, $response, $batch) {
    echo "Request $key failed: " . $response->message() . "\n";
})
->finally(function ($batch) {
    echo "Batch finished.\n";
})
->send();
```

### Batch Methods

| Method                 | Description                                |
| :--------------------- | :----------------------------------------- |
| `getRequests(): array` | Returns all configured requests            |
| `getResults(): array`  | Returns all response objects keyed by name |
| `hasFailed(): bool`    | Returns whether any request failed         |

## Response Object

The `send()` method returns a `Helpers\Http\Client\Response` instance.

#### Status Checks

| Method                     | Description                                    |
| :------------------------- | :--------------------------------------------- |
| `httpCode(): int`          | HTTP status code (200, 404, 500, etc.)         |
| `ok(): bool`               | True if status is exactly 200                  |
| `successful(): bool`       | True if status is 2xx                          |
| `failed(): bool`           | True if status >= 400 or transport error       |
| `clientError(): bool`      | True if status is 4xx                          |
| `serverError(): bool`      | True if status is 5xx                          |
| `isRedirect(): bool`       | True if status is 3xx                          |
| `notFound(): bool`         | True if status is 404                          |
| `isTransportError(): bool` | True if cURL error occurred (no HTTP response) |

> [!NOTE]
> Most status check methods have `is` prefixed aliases (e.g., `isSuccessful()`, `isFailed()`, `isNotFound()`, `isOk()`).

#### Body Access

| Method                                          | Description                            |
| :---------------------------------------------- | :------------------------------------- |
| `body(): ?string`                               | Raw response body                      |
| `isEmpty(): bool`                               | True if body is empty                  |
| `json(): mixed`                                 | JSON-decoded body                      |
| `bodyJson(string $key, $default = null): mixed` | Get value from JSON using dot notation |

```php
$response = curl()->get($url)->send();

// Get nested value with default
$email = $response->bodyJson('user.profile.email', 'unknown@example.com');
```

#### Headers

| Method                         | Description                            |
| :----------------------------- | :------------------------------------- |
| `responseHeaders(): ?array`    | All response headers                   |
| `header(string $key): ?string` | Get specific header (case-insensitive) |
| `location(): ?string`          | Get `Location` header (for redirects)  |

#### Metadata

| Method                  | Description                      |
| :---------------------- | :------------------------------- |
| `message(): string`     | Error message or 'Unknown error' |
| `transportCode(): int`  | cURL errno (0 = no error)        |
| `transferTime(): float` | Request duration in seconds      |

#### Error Handling

```php
onError(callable $callback): self
```

Executes callback if the request failed.

```php
$response = curl()->get($url)->send()
    ->onError(function ($response) {
        Log::error('Request failed', [
            'code' => $response->httpCode(),
            'message' => $response->message()
        ]);
    });
```

## Promise Object

The `Promise` class represents a deferred HTTP request.

| Method             | Description                                      |
| :----------------- | :----------------------------------------------- |
| `wait(): Response` | Blocks until request completes, returns Response |

```php
// Start multiple async requests
$promise1 = curl()->get($url1)->async();
$promise2 = curl()->get($url2)->async();

// Do other work...
processLocalData();

// Wait for results
$response1 = $promise1->wait();
$response2 = $promise2->wait();
```

---

#### Other Examples

**JSON POST with Retries**

```php
$response = curl()
    ->asJson()
    ->retry(3, 500)
    ->withToken($apiKey)
    ->post('https://api.service.com/data', ['id' => 1])
    ->send();

if ($response->successful()) {
    $result = $response->json();
}
```

**File Upload with Progress Tracking**

```php
$response = curl()
    ->attach('photo', storage_path('photos/avatar.jpg'), 'image/jpeg')
    ->post('https://api.service.com/upload', ['type' => 'avatar'])
    ->send();
```

**Parallel API Calls**

```php
$batch = Curl::pool(fn() => [
    'user' => curl()->withToken($token)->get("https://api.com/users/{$id}"),
    'orders' => curl()->withToken($token)->get("https://api.com/users/{$id}/orders"),
    'notifications' => curl()->withToken($token)->get("https://api.com/users/{$id}/notifications"),
])
->progress(function ($key, $response) {
    Log::info("Fetched {$key}");
})
->send();

$user = $batch->getResults()['user']->json();
$orders = $batch->getResults()['orders']->json();
```

**Error Handling Pattern**

```php
$response = curl()
    ->timeout(5000)
    ->retry(2, 200)
    ->get('https://api.unreliable.com/data')
    ->send();

if ($response->isTransportError()) {
    // Network/DNS/Timeout error
    Log::critical('Transport error: ' . $response->message());
} elseif ($response->serverError()) {
    // 5xx error
    Log::error('Server error: ' . $response->httpCode());
} elseif ($response->clientError()) {
    // 4xx error
    if ($response->notFound()) {
        throw new NotFoundException();
    }
    Log::warning('Client error: ' . $response->httpCode());
} else {
    // Success
    $data = $response->json();
}
```

## Related

- [Request](request-helper.md) - Handling incoming server requests
- [Response](response-helper.md) - Standard framework HTTP responses
- [Functions Reference](../functions.md#curl) - Global `curl()` helper
