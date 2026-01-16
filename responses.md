# Responses

The `Helpers\Http\Response` class helps you build and send HTTP responses with various content types, headers, and status codes.

## Basic Responses

In a controller, you can return strings, arrays, or Response objects.

````php
#### body
```php
body(string $content): self
````

Sets the raw content body of the response.

- **Example**: `return $this->response->body('Success')->status(200);`.

#### json

```php
json(mixed $data, int $status = 200, int $option = 0): self
```

Sends a JSON response and sets the `Content-Type` header to `application/json`. You can optionally pass `JSON_*` constants as the third parameter.

- **Example**: `return $this->response->json(['id' => 1]);`.

#### created

```php
created(?string $location = null): self
```

Sets a 201 Created status. If a `$location` is provided, it automatically sets the `Location` header.

- **Example**: `return $this->response->created('/api/users/1');`.

#### redirect / back

```php
redirect(string $url, int $status = 302): self
back(int $status = 302): self
```

Redirects the user to a new URL or back to the previous page.

- **Example**: `return $this->response->redirect(url('login'));`.

## Custom Responses

Build custom responses with headers and status codes.

```php
// Unauthorized response
return $this->response
    ->body('Unauthorized')
    ->status(401)
    ->header(['WWW-Authenticate' => 'Bearer']);

// No content response
return $this->response->status(204);

// Custom content type
return $this->response
    ->body($xmlContent)
    ->header(['Content-Type' => 'application/xml'])
    ->status(200);
```

#### download / file

```php
download(string $path, ?string $name = null): void
file(string $path, bool $inline = false, ?string $name = null): void
```

Streams a file to the browser. **Note**: These methods are terminal—they send the response and exit the script automatically.

- **Example**: `return $this->response->download($path, 'invoice.pdf');`.

## View Responses

```php
// Render a view (using Controller helper)
return $this->asView('welcome', ['name' => 'John']);

// View with status code (Response object doesn't handle views directly)
// You would typically handle status in the view logic or response modifications
// For simple view rendering:
return $this->asView('errors/404');
```

#### header / status

```php
header(array $headers): self
status(int $code): self
```

Fluently sets the HTTP status code and response headers.

- **Example**: `return $this->response->status(401)->header(['X-Reason' => 'Expired']);`.

## Status Codes

```php
// Success responses
return $this->response->json($data)->status(200); // OK
return $this->response->json($created)->status(201); // Created
return $this->response->status(204); // No Content

// Client error responses
return $this->response->json($error)->status(400); // Bad Request
return $this->response->json($error)->status(401); // Unauthorized
return $this->response->json($error)->status(403); // Forbidden
return $this->response->json($error)->status(404); // Not Found

// Server error responses
return $this->response->json($error)->status(500); // Internal Server Error
return $this->response->json($error)->status(503); // Service Unavailable
```

## Response Macros

You can extend the Response class with custom methods using macros:

```php
use Helpers\Http\Response;

// Register a macro
Response::macro('success', function($data) {
    return $this->json(['success' => true, 'data' => $data]);
});

// Use the macro
return $this->response->success($users);
```

## Content Negotiation

```php
// Respond based on Accept header
if ($this->request->wantsJson()) {
    return $this->asJson($data);
}

return $this->asView('users.index', ['users' => $data]);
```

## Convenience Methods

The Response class includes fluent helpers for common response types:

```php
// 200 OK
return $this->response->ok('Success');

// 201 Created (with optional location header)
return $this->response->created('/api/users/1');

// 403 Forbidden
return $this->response->forbidden('Access Denied');

// 404 Not Found
return $this->response->notFound('Page not found');
```

**Redirect Helpers:**

```php
// 301 Moved Permanently
return $this->response->permanentRedirect('/new-url');

// 303 See Other
return $this->response->seeOther('/resource');
```

## Status Inspection

You can inspect the status code of a response object:

```php
$response = $this->response->status(404);

if ($response->isClientError($response->getStatusCode())) {
    // Handle client error (4xx)
}

if ($response->isServerError($response->getStatusCode())) {
    // Handle server error (5xx)
}

if ($response->isSuccessful($response->getStatusCode())) {
    // Handle success (2xx)
}

if ($response->isRedirect($response->getStatusCode())) {
    // Handle redirect (3xx)
}

if ($response->isInformational($response->getStatusCode())) {
    // Handle info (1xx)
}

if ($response->isError($response->getStatusCode())) {
    // Handle any error (4xx or 5xx)
}
```

#### reason

```php
reason(int $status): string
```

Returns the human-readable status text for a given status code.

- **Example**: `$this->response->reason(404); // "Not Found"`
