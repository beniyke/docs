# Requests

The `Helpers\Http\Request` class provides a comprehensive interface for working with HTTP requests in your application.

## Accessing Request Data

#### all

```php
all(): array
```

Returns all input data (GET, POST, JSON, etc.) as a single associative array.

- **Example**: `$data = $request->all();`.

#### post / get

```php
post(?string $key = null, mixed $default = null): mixed
get(?string $key = null, mixed $default = null): mixed
```

Retrieves specific input from the POST body (including JSON) or the query string. Supports **dot notation** for accessing nested array data.

- **Example**: `$name = $request->post('user.profile.name', 'Anonymous');`.

### Nested Data Access

Anchor supports **dot notation** across all retrieval methods (`post`, `get`, `cookies`, `server`). This allows you to safely access deep array structures without manual checks.

```php
// If POST data is: ['user' => ['profile' => ['name' => 'John']]]
$name = request()->post('user.profile.name'); // 'John'

// With a default value
$theme = request()->get('settings.theme', 'light');
```

#### only / exclude

```php
only(array $keys): array
exclude(array $keys): array
```

Retrieves a subset of the request data or all data except the specified keys.

- **Use Case**: Filtering data before mass-assigning to a model or service.
- **Example**: `$data = $request->only(['email', 'password']);`.

## Checking Input Existence

### Check if Key Exists

```php
if ($request->has('email')) {
    // Email field exists in request
}

// Check multiple keys
if ($request->has(['email', 'password'])) {
    // Both fields exist
}
```

### Check if Field is Filled

```php
if ($request->filled('email')) {
    // Email exists and is not empty
}

// Check multiple fields
if ($request->filled(['name', 'email'])) {
    // Both fields are filled
}
```

### Check if Any Field is Filled

```php
if ($request->anyIsFilled(['phone', 'mobile', 'email'])) {
    // At least one contact method is provided
}
```

## Request Methods

### Checking Request Method

```php
if ($request->isPost()) {
    // Handle POST request
}

if ($request->isGet()) {
    // Handle GET request
}

if ($request->isPut()) {
    // Handle PUT request
}

if ($request->isPatch()) {
    // Handle PATCH request
}

if ($request->isDelete()) {
    // Handle DELETE request
}

// Check if method changes state
if ($request->isStateChanging()) {
    // POST, PUT, PATCH, or DELETE
}
```

#### method

```php
method(): string
```

Returns the HTTP request method in uppercase.

- **Example**: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`.

#### isOptions

```php
if ($request->isOptions()) {
    // Handle OPTIONS request
}
```

## File Uploads

#### hasFile

```php
if ($request->hasFile()) {
    // Request contains uploaded files
}
```

#### file

```php
file(?string $key = null): UploadedFile|array|null
```

Retrieves one or all uploaded files from the request. Returns an `UploadedFile` object for easy manipulation.

- **Example**: `$request->file('avatar')->move('/uploads');`.

### UploadedFile (FileHandler) Methods

When you retrieve a file using `$request->file()`, you get a `FileHandler` instance with several useful methods:

```php
$file = $request->file('avatar');

if ($file->isValid()) {
    $name = $file->getClientOriginalName();
    $extension = $file->getExtension();
    $size = $file->getSize(); // in bytes
    $mimeType = $file->getClientMimeType();
} else {
    $error = $file->getErrorMessage();
}
```

### Multiple File Uploads

```php
$files = $request->file('documents');

if (is_array($files)) {
    foreach ($files as $file) {
        $file->move('/path/to/destination');
    }
}
```

## Headers and Cookies

### Accessing Headers

```php
// Get specific header
$contentType = $request->header('content-type');

// Get all headers
$headers = $request->header();

// Check authorization header
$token = $request->getBearerToken();
$authHeader = $request->getAuthToken();
```

### Accessing Cookies

```php
// Get specific cookie
$sessionId = $request->cookies('session_id');

// Get all cookies
$cookies = $request->cookies();
```

### Server Variables

```php
$userAgent = $request->server('HTTP_USER_AGENT');
$remoteAddr = $request->server('REMOTE_ADDR');
```

## Request Information

### URL and Routing

```php
// Get current URI
$uri = $request->uri();

// Get full URL
$url = $request->baseUrl();

// Get domain
$domain = $request->domain();

// Get host
$host = $request->host();

// Get scheme
$scheme = $request->scheme(); // 'http' or 'https'

// Check if HTTPS
if ($request->secure()) {
    // Request is over HTTPS
}
```

### User Agent and IP

```php
// Get user agent string
$userAgent = $request->userAgent();

// Get client IP address
$ip = $request->ip();
```

### Request Type Detection

```php
// Check if AJAX request
if ($this->request->isAjax()) {
    return $this->asJson(['data' => $data]);
}

// Check content type
if ($this->request->contentTypeIs('json')) {
    // Request content type is application/json
}

// Check accepted response type
if ($this->request->wantsJson()) {
    return $this->asJson($data);
}

if ($this->request->accepts(['html', 'json'])) {
    // Request accepts either HTML or JSON
}

// Check if bot (honeypot triggered)
if ($this->request->isBot()) {
    // Block request
}

// Check if request passes security validation
// Includes CSRF, Honeypot, and Robot check
if ($this->request->isSecurityValid()) {
    // Safe to proceed
}
```

## Authenticated User

If the request has been processed by an authentication layer, you can access the user and their token directly:

```php
// Get the authenticated user object
$user = $request->user();

// Get the raw authentication token
$token = $request->token();
```

## Request Macros

You can extend the Request class with custom methods using macros:

```php
use Helpers\Http\Request;

// Register a macro
Request::macro('isAdmin', function() {
    return $this->user() && $this->user()->role === 'admin';
});

// Use the macro
if ($this->request->isAdmin()) {
    // ...
}
```

## Security Features

### CSRF Protection

```php
// Get CSRF token
$token = $request->getCsrfToken();

// Check if CSRF token is valid (automatic in framework)
if ($request->csrfTokenIsValid()) {
    // Token is valid
}

// Check if request passes security validation
if ($request->isSecurityValid()) {
    // CSRF valid, not a bot, not a robot
}
```

### Referer

```php
$referer = $request->referer();
```

## Input Sanitization

By default, all input is sanitized. You can disable this:

```php
// Disable sanitization
$request->sanitize(false);
$rawData = $request->all();

// Re-enable sanitization
$request->sanitize(true);
$cleanData = $request->all();
```

## JSON and XML Requests

The Request class automatically parses JSON and XML payloads:

### JSON Requests

```php
// Client sends: {"name": "John", "email": "john@example.com"}
// Content-Type: application/json

$name = $request->post('name'); // 'John'
$email = $request->post('email'); // 'john@example.com'
```

### XML Requests

```php
// Client sends XML
// Content-Type: application/xml

$data = $request->post();
// Automatically parsed from XML to array
```

## PSR-7 Compliance

The Request class is PSR-7 compliant:

```php
// Get URI object
$uri = $request->getUri();

// Get protocol version
$version = $request->getProtocolVersion();

// Get request target
$target = $request->getRequestTarget();

// Get headers (PSR-7 format)
$headers = $request->getHeaders();

// Check if header exists
if ($request->hasHeader('Authorization')) {
    $auth = $request->getHeader('Authorization');
}

// Get header line
$contentType = $request->getHeaderLine('Content-Type');
```

### Immutable Request Modifications

```php
// Create new request with different method
$newRequest = $request->withMethod('PUT');

// Create new request with different URI
$newRequest = $request->withUri($uri);

// Add header
$newRequest = $request->withHeader('X-Custom', 'value');

// Add additional header value
$newRequest = $request->withAddedHeader('Accept', 'application/json');
```

## Route Helpers

The Request class provides several helpers to identify the current route and its properties:

```php
// Check if the current route is an API route
if ($request->routeIsApi()) {
    // ...
}

// Check if current route is the login or logout route
if ($request->isLoginRoute()) {
    // ...
}

if ($request->isLogoutRoute()) {
    // ...
}

// Check if the route requires authentication (based on config)
if ($request->shouldRequireAuth()) {
    // ...
}

// Get the route of the referer (previous page)
$prevRoute = $request->refererRoute();
```

## Route Metadata

```php
// Get current route
$route = $request->route();

// Get route by name
$loginRoute = $request->routeName('auth.login');

// Get full URL for route
$url = $request->fullRoute('users/profile');

// Get full URL by route name
$url = $request->fullRouteByName('users.show');

// Get callback route
$callback = $request->callback();
```

## Real-World Examples

### Form Validation

```php
public function store()
{
    // Check required fields
    if (!$this->request->filled(['name', 'email', 'password'])) {
        $this->flash->error('All fields are required');
        return $this->response->back();
    }

    // Get only needed fields
    $data = $this->request->only(['name', 'email', 'password']);

    // Create user
    User::create($data);
}
```

### File Upload Handling

```php
public function uploadAvatar()
{
    if (!$this->request->file('avatar')) {
        return $this->asJson(['error' => 'No file uploaded'], 400);
    }

    $file = $this->request->file('avatar');

    // Validate and move securely
    // This will validate type (image), size (2MB), and generate a safe filename
    $path = $file->moveSecurely('/uploads/avatars', [
        'type' => 'image',
        'maxSize' => 2097152
    ], false);

    if (empty($path)) {
        return $this->asJson(['error' => $file->getValidationError()], 400);
    }

    return $this->asJson(['filename' => basename($path)]);
}
```

### API Request Handling

```php
public function apiEndpoint()
{
    // Check if JSON request
    if ($this->request->contentTypeIs('json')) {
        $data = $this->request->post();

        // Validate bearer token
        $token = $this->request->getBearerToken();
        if (!$token) {
            return $this->asJson(['error' => 'Unauthorized'], 401);
        }

        // Process request
        return $this->asJson(['success' => true, 'data' => $data]);
    }

    return $this->asJson(['error' => 'Invalid content type'], 400);
}
```

### AJAX Response

```php
public function search()
{
    $query = $this->request->get('q');

    $results = Product::where('name', 'LIKE', "%{$query}%")->get();

    if ($this->request->isAjax()) {
        return $this->asJson($results->all());
    }

    return $this->asView('search/results', ['results' => $results]);
}
```

## Best Practices

- **Always validate input**: Never trust user input
- **Use `only()` or `exclude()`**: Be explicit about what data you accept
- **Check `filled()` for required fields**: Ensures fields exist and aren't empty
- **Sanitize when needed**: Default sanitization is enabled, disable only when necessary
- **Use type checking**: Validate file types and sizes before processing
- **Leverage security features**: CSRF protection is automatic for state-changing requests
