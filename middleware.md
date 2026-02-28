# Middleware

Middleware provides a convenient mechanism for filtering and inspecting HTTP requests entering your application. Middleware can perform tasks before or after passing the request to the next layer.

## How Middleware Works

Middleware forms a pipeline around your application. Each middleware:

- Receives the `Request` and `Response` objects
- Performs its logic
- Calls `$next($request, $response)` to pass control to the next middleware
- Can modify the request/response before or after calling `$next()`

## Creating Middleware

All middleware must implement `Core\Middleware\MiddlewareInterface`:

### Using CLI

You can generate a new middleware using the CLI:

```bash
# Generate Web Middleware (App/Middleware/Web)
# Creates CheckAgeMiddleware.php
php dock middleware:create CheckAge --web

# Generate API Middleware (App/Middleware/Api)
# Creates ValidateApiKeyMiddleware.php
php dock middleware:create ValidateApiKey --api
```

> The `Middleware` suffix is automatically appended to the class name if not provided. `CheckAge` becomes `CheckAgeMiddleware`.

### Deleting Middleware

```bash
php dock middleware:delete CheckAge --web
```

> You will be prompted to confirm deletion before the file is removed.

### Manual Creation

#### `handle` Method

```php
public function handle(Request $request, Response $response, Closure $next): mixed
```

The core of every middleware. It receives the current request and response objects, and a `$next` closure to continue the pipeline.

- **Before Logic**: Code placed before `$next($request, $response)` executes as the request enters the application.
- **After Logic**: Code placed after `$next($request, $response)` executes as the response leaves the application.
- **Termination**: Returning a `Response` directly (without calling `$next`) stops the pipeline immediately.

## Registering Middleware

Middleware is configured in `App/Config/middleware.php`. 

> [!NOTE]
> Core framework middlewares (`Session`, `Firewall`, `SmartValidation`) are now automatically injected by the system. You only need to register your application-specific middlewares here.

```php
return [
    'web' => [
        App\Middleware\Web\RedirectIfAuthenticatedMiddleware::class,
        App\Middleware\Web\WebAuthMiddleware::class,
        App\Middleware\Web\PasswordUpdateMiddleware::class,
        Debugger\Middleware\DebuggerMiddleware::class
    ],
    'api' => [
        App\Middleware\Api\ApiAuthMiddleware::class,
    ],
];
```

## Middleware Groups

Middleware is organized into groups (e.g., `web`, `api`). Routes are assigned to groups based on patterns:

```php
// Routes matching these patterns use 'web' middleware
'auth' => [
    'web' => ['auth/{*}', 'account/{*}'],
    'api' => ['api/{*}'],
],
```

### System Middleware

Anchor automatically loads essential middlewares for the `web` and `api` groups. These are defined in the `Kernel` and execute before your application middlewares.

#### SessionMiddleware (Web Only)

Manages the session lifecycle. In Anchor 2.6.0+, the framework is **stateless by default**. Sessions are only started when this middleware is invoked.

- **Automatic Start**: `session->start()` is called automatically for the `web` group.
- **Maintenance**: Handles periodic ID regeneration for security.

#### FirewallMiddleware (Web & API)

Provides advanced security filtering before the request reaches the application core.

- **Automatic Application**: Applied to both `web` and `api` groups by default.
- **Use Case**: IP whitelisting, brute-force protection, and malicious user-agent blocking.

#### SmartValidationMiddleware (Web & API)

Detects and executes [Request Validations](validation.md) based on convention.

- **Automatic Discovery**: Automatically looks for validation classes matching the current controller and action.

Handles authentication and authorization for web-based routes.

**Use Case**: Redirecting unauthenticated users to the login page or blocking access to specific administrative URLs.

**Example**: `return $response->redirect($request->fullRouteByName('login'))`.

```php
// Check authentication and authorization
if (!$this->auth->isAuthenticated() || !$this->auth->isAuthorized($request->route())) {
    $this->auth->logout();
    $loginRoute = $request->getRouteContext('login_route') ?? 'login';
    return $response->redirect($request->fullRouteByName($loginRoute));
}

// Multi-Guard Support
// You can define multiple guards for a route to support different user types
$guards = $request->getRouteContext('guards') ?? ['web'];
foreach ($guards as $guard) {
    if ($this->auth->viaGuard($guard)->isAuthenticated()) {
        // Find and set the user for the request
        $request->setAuthenticatedUser($this->auth->user());
        $request->setRouteContext('auth_guard', $guard);
        break;
    }
}

return $next($request, $response);
```

### RedirectIfAuthenticatedMiddleware

Redirects authenticated users away from guest-only pages (like login):

```php
class RedirectIfAuthenticatedMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        if ($this->auth->isAuthenticated() && $request->isLoginRoute()) {
            return $response->redirect($request->fullRouteByName('home'));
        }

        return $next($request, $response);
    }
}
```

### PasswordUpdateMiddleware

Forces password update for users with expired passwords:

```php
class PasswordUpdateMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        $user = $this->auth->user();

        if ($user && $user->passwordNeedsUpdate() && !$request->isPasswordUpdateRoute()) {
            return $response->redirect($request->fullRouteByName('change-password'));
        }

        return $next($request, $response);
    }
}
```

### FirewallMiddleware

Provides advanced security filtering before the request even reaches the application core.

- **Use Case**: Implementing IP whitelisting, protecting against brute-force attacks, or blocking known malicious user agents.

### Security Headers

Adds security headers to all responses to protect against common attacks:

```php
class SecurityHeadersMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        $response = $next($request, $response);

        // Add security headers
        $response->header([
            'X-Frame-Options' => 'SAMEORIGIN',
            'X-Content-Type-Options' => 'nosniff',
            'X-XSS-Protection' => '1; mode=block',
            'Referrer-Policy' => 'strict-origin-when-cross-origin'
        ]);

        // HSTS (only on HTTPS)
        if ($request->isSecure()) {
            $response->header(['Strict-Transport-Security' => 'max-age=31536000; includeSubDomains']);
        }

        return $response;
    }
}
```

**Headers Added:**

- `X-Frame-Options: SAMEORIGIN` - Prevents clickjacking
- `X-Content-Type-Options: nosniff` - Prevents MIME sniffing
- `X-XSS-Protection: 1; mode=block` - Legacy XSS protection
- `Referrer-Policy` - Controls referrer information
- `Permissions-Policy` - Controls browser features
- `Strict-Transport-Security` - HSTS for HTTPS

**Configuration:** `App/Config/default.php` → `security_headers`

**Usage:**

```php
// Add to middleware stack
'middlewares' => [
    'web' => [
        App\Middleware\Web\SecurityHeadersMiddleware::class,
        // ... other middleware
    ],
],
```

### CorsMiddleware

Handles Cross-Origin Resource Sharing (CORS) for API endpoints:

```php
class CorsMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        // Handle preflight OPTIONS request
        if ($request->isOptions()) {
            return $response->header([
                'Access-Control-Allow-Origin' => '*',
                'Access-Control-Allow-Methods' => 'GET, POST, PUT, DELETE, OPTIONS',
                'Access-Control-Allow-Headers' => 'Content-Type, Authorization'
            ])->status(204);
        }

        $response = $next($request, $response);
        $response->header(['Access-Control-Allow-Origin' => '*']);

        return $response;
    }
}
```

**Features:**

- Preflight request handling
- Origin validation (including wildcards like `*.example.com`)
- Configurable allowed methods and headers
- Credentials support

**Configuration:** `App/Config/cors.php`

**Usage:**

````php
// Add to API middleware
```php
return [
    'api' => [
        App\Middleware\Api\CorsMiddleware::class,
        // ... other middleware
    ],
];
````

## Middleware Execution Order

Middleware executes in a specific order: **System Middleware** first, followed by **Application Middleware** in the order defined in your configuration.

### Default Web Order

1.  **`SessionMiddleware`** (System) - Starts session
2.  **`FirewallMiddleware`** (System) - Checks security rules
3.  **`SmartValidationMiddleware`** (System) - Validates incoming data
4.  **`RedirectIfAuthenticatedMiddleware`** (App)
5.  **`WebAuthMiddleware`** (App)
6.  **`PasswordUpdateMiddleware`** (App)

### Default API Order

1.  **`FirewallMiddleware`** (System)
2.  **`SmartValidationMiddleware`** (System)
3.  **`ApiAuthMiddleware`** (App)

**Request Flow**:

- Request enters first middleware
- Each middleware calls `$next()` to pass to next
- Request reaches controller
- Response flows back through middleware in reverse
- Response sent to client

## Creating Custom Middleware

### Logging Middleware

```php
namespace App\Middleware\Web;

use Core\Middleware\MiddlewareInterface;
use Closure;
use Helpers\Http\Request;
use Helpers\Http\Response;
use Helpers\Log;

class LoggingMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        // Log request
        Log::channel('requests')->info('Request: ' . $request->method() . ' ' . $request->uri());

        // Pass to next middleware
        $result = $next($request, $response);

        // Log response
        Log::channel('requests')->info('Response: ' . $response->getStatusCode());

        return $result;
    }
}
```

### CORS Middleware

```php
namespace App\Middleware\Api;

use Core\Middleware\MiddlewareInterface;
use Closure;
use Helpers\Http\Request;
use Helpers\Http\Response;

class CorsMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        // Handle preflight requests
        if ($request->isOptions()) {
            return $response->header([
                'Access-Control-Allow-Origin' => '*',
                'Access-Control-Allow-Methods' => 'GET, POST, PUT, DELETE, OPTIONS',
                'Access-Control-Allow-Headers' => 'Content-Type, Authorization',
            ])->status(204);
        }

        // Add CORS headers to response
        $result = $next($request, $response);

        $response->header([
            'Access-Control-Allow-Origin' => '*',
        ]);

        return $result;
    }
}
```

### Rate Limiting Middleware

```php
namespace App\Middleware\Api;

use Core\Middleware\MiddlewareInterface;
use Closure;
use Helpers\Http\Request;
use Helpers\Http\Response;

class RateLimitMiddleware implements MiddlewareInterface
{
    private const MAX_REQUESTS = 60;
    private const WINDOW_SECONDS = 60;

    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        $key = 'rate_limit:' . $request->ip();
        $cache = cache('rate_limits');

        $requests = (int) $cache->read($key) ?? 0;

        if ($requests >= self::MAX_REQUESTS) {
            return $response
                ->json(['error' => 'Too many requests'])
                ->status(429)
                ->header(['Retry-After' => (string) self::WINDOW_SECONDS]);
        }

        $cache->write($key, $requests + 1, self::WINDOW_SECONDS);

        return $next($request, $response);
    }
}
```

## Dependency Injection

Middleware supports constructor dependency injection:

```php
class MyMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly UserService $userService,
        private readonly ConfigServiceInterface $config,
        private readonly Session $session
    ) {}

    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        // Use injected dependencies
        $user = $this->userService->find($this->session->get('user_id'));

        return $next($request, $response);
    }
}
```

## Terminating Middleware

Middleware can terminate the request without calling `$next()`:

```php
public function handle(Request $request, Response $response, Closure $next): mixed
{
    if (!$this->isAllowed($request)) {
        // Terminate - don't call $next()
        return $response
            ->status(403)
            ->body('Access denied');
    }

    return $next($request, $response);
}
```

## Modifying Requests

```php
public function handle(Request $request, Response $response, Closure $next): mixed
{
    // Modify request before passing forward
    $request->sanitize(true);

    return $next($request, $response);
}
```

## Modifying Responses

```php
public function handle(Request $request, Response $response, Closure $next): mixed
{
    // Get response from next middleware/controller
    $result = $next($request, $response);

    // Modify response
    $response->header(['X-Custom-Header' => 'Value']);

    return $result;
}
```

## Best Practices

- **Keep middleware focused**: Each middleware should do one thing
- **Order matters**: Place authentication before authorization
- **Use dependency injection**: Don't create dependencies manually
- **Handle errors gracefully**: Return appropriate responses
- **Consider performance**: Middleware runs on every request
- **Document behavior**: Explain what your middleware does
- **Test thoroughly**: Middleware affects all routes

## Common Use Cases

- **Authentication**: Verify user is logged in
- **Authorization**: Check user permissions
- **Logging**: Record requests and responses
- **CORS**: Handle cross-origin requests
- **Rate Limiting**: Prevent abuse
- **Session Management**: Handle sessions
- **Security Headers**: Add security headers
- **Request Validation**: Validate input
- **Response Transformation**: Modify responses
- **Maintenance Mode**: Block access during maintenance
