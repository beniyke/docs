# Authentication

Anchor provides authentication through the `AuthService` and session management system.

## Configuration

Authentication settings are defined in:

- `App/Config/default.php` - Session configuration
- `App/Config/firewall.php` - Firewall and security settings
- `App/Config/route.php` - Protected routes

## How Authentication Works

Anchor uses **session token-based authentication**:

- User logs in with credentials
- `WebAuthService` validates credentials via `UserService`
- `SessionService` creates a session record in database with unique token
- Session token is stored in user's session
- On subsequent requests, token is validated against database
- Session is refreshed on each valid request

## The AuthService

Controllers access authentication through `$this->auth` (instance of `AuthServiceInterface`).

#### isAuthenticated

```php
isAuthenticated(): bool
```

Checks if the current user has a valid, active session.

- **Use Case**: Conditionally displaying navigation links like "My Account" vs "Login".
- **Example**: `if ($this->auth->isAuthenticated()) { ... }`.

#### user

```php
user(): ?User
```

Retrieves the currently authenticated `User` model instance. Returns `null` if the user is not logged in.

- **Use Case**: Accessing the logged-in user's profile data or ID for database queries.

#### login

```php
login(LoginRequest $request): bool
```

Attempts to authenticate a user using the provided request object. It handles credential verification, session creation, and success/error flash messages.

- **Use Case**: Handling a POST request from the login form.

#### logout

```php
logout(?string $session_token = null): bool
```

Terminates the current user's session, destroys the PHP session, and removes the session record from the database.

- **Example**: `return $this->auth->logout() ? redirect('/') : back();`.

#### isAuthorized

```php
isAuthorized(string $route): bool
```

Checks if the authenticated user has permission to access the specified logical route.

- **Use Case**: Protecting specific administrative or restricted sub-pages within a module.

## Logging In

The login flow uses `WebAuthService`:

```php
namespace App\Auth\Controllers;

use App\Auth\Validations\Form\LoginFormRequestValidation;
use App\Core\BaseController;
use Helpers\Http\Response;

class LoginController extends BaseController
{
    public function attempt(LoginFormRequestValidation $validator): Response
    {
        // Validate request
        $validator->validate($this->request->post());

        if ($validator->has_error()) {
            $this->flash->error($validator->errors());
            return $this->response->redirect($this->request->fullRoute());
        }

        // Attempt login via AuthService
        if (!$this->auth->login($validator->getRequest())) {
            // Login failed - flash message already set by service
            return $this->response->redirect($this->request->fullRoute());
        }

        // Success - redirect to home
        return $this->response->redirect($this->request->fullRouteByName('home'));
    }
}
```

### What Happens During Login

The `WebAuthService->login()` method:

- Validates credentials using `UserService->confirmUser()`
- Verifies password with `SymmetricEncryptor->verifyPassword()`
- Creates session via `SessionService->createNewSession()`
- Stores session token in user's session
- Clears firewall failed attempts
- Shows success flash message
- Sends login notification via `LoginInAppNotification`

## Retrieving the Authenticated User

```php
// In a controller
$user = $this->auth->user();

if ($user) {
    echo $user->name;
    echo $user->email;
}
```

The `user()` method:

- Gets session token from session storage
- Looks up session in database via `SessionService`
- Validates session is not expired
- Refreshes session expiry
- Returns associated `User` model

## Checking Authentication

```php
// In controller
if ($this->auth->isAuthenticated()) {
    // User is logged in
}

// In middleware
if (!$this->auth->isAuthenticated()) {
    return $response->redirect($request->fullRouteByName('login'));
}
```

## Checking Authorization

```php
// Check if user can access a specific route
if ($this->auth->isAuthorized('account/settings')) {
    // User has access
}
```

Authorization checks:

- User is authenticated
- User can login (`$user->canLogin()`)
- Route is in user's accessible routes (via `MenuService`)

## Logging Out

```php
public function logout(): Response
{
    $this->auth->logout();
    return $this->response->redirect($this->request->fullRouteByName('login'));
}
```

The `logout()` method:

- Gets session token from session
- Deletes session token from session storage
- Destroys PHP session
- Terminates database session record

## Password Hashing

Use the `enc()` helper (which uses `SymmetricEncryptor`) for password hashing:

```php
// Hash a password
$hashedPassword = enc()->hashPassword('user-password');

// Verify a password
if (enc()->verifyPassword('user-password', $hashedPassword)) {
    // Password is correct
}
```

Passwords are hashed using **Argon2ID** algorithm.

## Protecting Routes

Configure protected routes in `App/Config/route.php`:

```php
'auth' => [
    'web' => ['auth/{*}', 'account/{*}'],
    'api' => ['api/{*}'],
],
```

Routes matching these patterns require authentication.

### Excluding Routes from Authentication

```php
'auth-exclude' => [
    'web' => [
        'auth/{login, recoverpassword, resetpassword, signup, activation}',
    ],
    'api' => [],
],
```

## Authentication Middleware

The `WebAuthMiddleware` protects routes:

```php
namespace App\Middleware\Web;

use App\Services\Auth\Interfaces\AuthServiceInterface;
use Helpers\Http\Request;
use Helpers\Http\Response;

class WebAuthMiddleware
{
    public function __construct(
        private readonly AuthServiceInterface $auth
    ) {}

    public function handle(Request $request, Response $response, Closure $next): mixed
    {
        // Check if route should bypass auth
        if ($request->routeShouldBypassAuth()) {
            return $next($request, $response);
        }

        // Check authentication and authorization
        if (!$this->auth->isAuthenticated() || !$this->auth->isAuthorized($request->route())) {
            $this->auth->logout();
            return $response->redirect(url('login'));
        }

        return $next($request, $response);
    }
}
```

## Session Management

Sessions are managed through the `SessionService`:

- Sessions are stored in the database (`session` table)
- Each session has a unique token
- Sessions have configurable lifetime
- "Remember me" extends session lifetime
- Sessions are automatically refreshed on valid requests
- Expired sessions are cleaned up

### Session Configuration

In `App/Config/default.php`:

```php
'timeout' => 3600, // 1 hour
'cookie' => [
    'remember_me_lifetime' => 2592000, // 30 days
],
```

## Firewall Protection

The Firewall system provides additional security through rate limiting and IP blocking. Failed login attempts are tracked and can temporarily block access.

**Example**

Complete login flow from actual codebase:

```php
// LoginController.php
public function attempt(LoginFormRequestValidation $validator): Response
{
    $validator->validate($this->request->post());

    if ($validator->has_error()) {
        $this->flash->error($validator->errors());
        return $this->response->redirect($this->request->fullRoute());
    }

    if (!$this->auth->login($validator->getRequest())) {
        return $this->response->redirect($this->request->fullRoute());
    }

    return $this->handleLoginRedirect();
}

// WebAuthService.php
public function login(LoginRequest $request): bool
{
    if (!$request->isValid()) {
        $this->firewall->fail()->capture();
        return false;
    }

    $user = $this->user_service->confirmUser($request->getData());

    if (!$user) {
        $this->flash->error('Invalid login credentials.');
        $this->firewall->fail()->capture();
        return false;
    }

    $should_remember = $request->hasRememberMe();
    $db_lifetime = $should_remember
        ? $this->config->get('session.cookie.remember_me_lifetime', 0)
        : $this->config->get('session.timeout');

    $session = $this->session_service->createNewSession($user, $db_lifetime);

    if (!$session) {
        $this->flash->error('Login failed. Please try again.');
        $this->firewall->fail()->capture();
        return false;
    }

    $this->session->regenerateId();
    $this->session->set($this->config->get('session.name'), $session->token);
    $this->firewall->clear()->capture();
    $this->flash->success('Welcome ' . $user->name);

    return true;
}
```

## Best Practices

- **Use AuthService methods**: Don't manually manage sessions
- **Validate before authenticating**: Always validate input first
- **Use flash messages**: Provide user feedback
- **Handle failures gracefully**: Check return values
- **Leverage firewall**: Let it handle throttling
- **Use "remember me" carefully**: Understand security implications
- **Clean up sessions**: Automatically handled by `SessionService` (configurable lottery)
