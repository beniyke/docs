# Response Class

The `Helpers\Http\Response` class provides a simple and flexible way to construct and send HTTP responses.

## Class Usage

### Direct Instantiation

```php
use Helpers\Http\Response;

$response = new Response('Hello, World!', 200);
return $response;
```

### Using Dependency Injection

```php
use Helpers\Http\Response;

class ApiController
{
    public function index()
    {
        $response = new Response();
        return $response->json(['status' => 'success']);
    }
}
```

### Helper Function (Convenience)

For quick access, use the `response()` helper:

```php
return response('Hello, World!');
return response()->json(['status' => 'success']);
```

> **Note**: The `response()` helper creates a new `Response` instance each time. The Response class doesn't need injection as it's typically created per-response.

## Basic Responses

### Text Responses

```php
use Helpers\Http\Response;

// Simple text
$response = new Response('Hello, World!');

// With status code
$response = new Response('Not Found', 404);

// With headers
$response = new Response('Success', 200, [
    'Content-Type' => 'text/plain',
    'X-Custom-Header' => 'value'
]);
```

## JSON Responses

#### json

```php
json(mixed $data, int $status = 200, int $options = 0): self
```

Converts an array or object into a JSON response and sets the `Content-Type` header to `application/json`.

- **Use Case**: Returning data to a frontend SPA or mobile application.
- **Example**: `return response()->json(['id' => 1, 'name' => 'John'])`.

## Status-Specific Responses

#### ok

```php
ok(mixed $content): self
```

Returns a standard 200 OK response with the provided content.

#### created

```php
created(string $location): self
```

Returns a 201 Created response and sets the `Location` header to the provided URI.

- **Use Case**: Successfully creating a new resource via a POST request.

#### notFound

```php
notFound(?string $message = null): self
```

Returns a 404 Not Found response.

- **Use Case**: When a requested database record or page does not exist.

#### forbidden

```php
forbidden(?string $message = null): self
```

Returns a 403 Forbidden response.

- **Use Case**: When a user is authenticated but lacks permission to access a resource.

## Redirects

#### redirect

```php
redirect(string $url, int $status = 302): self
```

Creates a redirect response to the specified URL.

- **Use Case**: Sending a user to a login page or dashboard after successful authentication.

#### permanentRedirect

```php
permanentRedirect(string $url): self
```

Creates a 301 Moved Permanently redirect.

- **Use Case**: When a page has been moved to a new URL and you want to preserve SEO.

#### seeOther

```php
seeOther(string $url): self
```

Creates a 303 See Other redirect.

- **Use Case**: Redirecting after a POST request to ensure the user doesn't re-submit the form on refresh.

### Redirect Back

```php
// Redirect to previous page
return (new Response())->back();
```

## File Responses

### File Downloads

```php
// Download a file
return (new Response())->download('/path/to/file.pdf');

// Download with custom filename
return (new Response())->download('/path/to/file.pdf', 'invoice.pdf');
```

### Display Files Inline

```php
// Display PDF in browser
return (new Response())->file('/path/to/document.pdf', true);

// Force download
return (new Response())->file('/path/to/document.pdf', false);

// With custom filename
return (new Response())->file('/path/to/document.pdf', true, 'report.pdf');
```

### Stream External Content

```php
// Serve external content
return (new Response())->content('https://example.com/image.jpg');
```

## Headers and Status

### Set Headers

```php
$response = new Response('Content');

// Set multiple headers
$response->header([
    'Cache-Control' => 'no-cache',
    'X-Custom' => 'value'
]);

return $response;
```

### Set Individual Header

```php
$response = new Response();
$response->setHeader('Content-Type', 'application/json');
```

### Set Status Code

```php
$response = new Response();

// Set status with reason
$response->status(201, 'Created');

// Set status only
$response->setStatusCode(404);

return $response;
```

### Set Body

```php
$response = new Response();
$response->body('Response content');

return $response;
```

## Method Chaining

```php
return (new Response())
    ->status(200)
    ->header(['Content-Type' => 'application/json'])
    ->body(json_encode(['success' => true]));
```

## Get Response Data

```php
$response = new Response('Content', 200, ['X-Custom' => 'value']);

// Get content
$content = $response->getContent();

// Get status code
$status = $response->getStatusCode();

// Get reason phrase
$reason = $response->getReasonPhrase();

// Get all headers
$headers = $response->getHeaders();

// Get specific header
$customHeader = $response->getHeader('X-Custom');
```

## Sending Responses

```php
$response = new Response('Hello');

// Send headers and content
$response->send();

// Send headers only
$response->sendHeaders();

// Send content only
$response->sendContent();
```

#### complete / end

```php
complete(?callable $callback = null): void
```

Sends the response and ends the script execution.

```php
end(?callable $callback = null): void
```

Ends the script execution without sending the response (useful if headers/content were already sent).

## Status Code Helpers

```php
use Helpers\Http\Response;

$response = new Response();

// Check status code types
$response->isInformational(100); // true (1xx)
$response->isSuccessful(200);     // true (2xx)
$response->isRedirect(301);       // true (3xx)
$response->isClientError(404);    // true (4xx)
$response->isServerError(500);    // true (5xx)
$response->isError(400);          // true (4xx or 5xx)
```

## Real-World Examples

### API Controller

```php
use Helpers\Http\Response;

class UserApiController
{
    public function index()
    {
        $users = $this->userService->getAll();

        return (new Response())->json([
            'success' => true,
            'data' => $users
        ]);
    }

    public function show(int $id)
    {
        $user = $this->userService->find($id);

        if (!$user) {
            return (new Response())->json([
                'error' => 'User not found'
            ], 404);
        }

        return (new Response())->json([
            'success' => true,
            'data' => $user
        ]);
    }
}
```

### File Download

```php
use Helpers\Http\Response;

class ReportController
{
    public function download(int $id)
    {
        $reportPath = $this->generateReport($id);

        return (new Response())->download(
            $reportPath,
            "report-{$id}.pdf"
        );
    }
}
```

### Custom Response Headers

```php
use Helpers\Http\Response;

class ApiController
{
    public function index()
    {
        $response = new Response();

        return $response
            ->header([
                'Cache-Control' => 'max-age=3600',
                'X-API-Version' => '1.0',
                'X-Rate-Limit' => '100'
            ])
            ->json(['data' => $results]);
    }
}
```

## Common Status Codes

| Code | Meaning               | Method                |
| ---- | --------------------- | --------------------- |
| 200  | OK                    | `ok()`                |
| 201  | Created               | `created()`           |
| 301  | Moved Permanently     | `permanentRedirect()` |
| 302  | Found (Temporary)     | `redirect()`          |
| 303  | See Other             | `seeOther()`          |
| 400  | Bad Request           | `status(400)`         |
| 401  | Unauthorized          | `status(401)`         |
| 403  | Forbidden             | `forbidden()`         |
| 404  | Not Found             | `notFound()`          |
| 500  | Internal Server Error | `status(500)`         |

## Best Practices

```php
use Helpers\Http\Response;

// Good - Use specific status methods
return (new Response())->notFound();

// Good - Chain methods for clarity
return (new Response())
    ->status(201)
    ->header(['Location' => '/users/123'])
    ->json(['id' => 123]);

// Good - Use JSON for API responses
return (new Response())->json(['data' => $results]);

// Avoid - Manual JSON encoding when json() exists
return (new Response())->body(json_encode($data)); // Use json() instead

// Good - Proper redirect status codes
return (new Response())->permanentRedirect($url); // 301
return (new Response())->redirect($url);          // 302
```

## Related

- [Request](../request-helper.md) - Handle incoming requests
- [Redirect](../redirect-helper.md) - Redirect helper
