# Authentication

Anchor provides a multi-driver, enterprise-grade authentication ecosystem. This guide covers how to implement, extend, and understand the technical foundations of the system.

## Quick Implementation

### Core Authentication Service (`AuthService`)

Anchor provides a single `AuthService` in the core layer that handles both Web and API flows. It returns an `AuthResult` object which provides a structured response.

```php
public function login(LoginRequest $request): AuthResult
{
    // The core service handles the complexity of guards and events
    return $this->auth->login($request);
}
```

In your controllers, you can handle the result:

```php
$result = $this->auth->login($request);

if ($result->failed()) {
    $this->flash->error($result->message());
    return;
}

$user = $result->user();
$this->flash->success('Welcome back!');
```

### `validate()` vs `attempt()`

Understanding the difference between these two methods (available via the `AuthManager`) is crucial:

- **`attempt(array $credentials)`**: **Stateful.** It verifies the credentials and, if successful, automatically logs the user in (e.g., creates a session or sets the user for the current request).
- **`validate(array $credentials)`**: **Stateless.** It only checks if the credentials are correct and returns a boolean. It does **not** log the user in. Use this for "Confirm Password" checks before sensitive actions.

## Extending the System

Anchor is designed to be highly extensible. You can swap or add components as your application grows.

### Custom Guards (e.g., JWT)

Implement the `Security\Auth\Interfaces\GuardInterface` and register it in a Service Provider:

```php
$manager->extend('jwt', function($name, $config) {
    return new JwtGuard($manager->resolveSource($config['source']));
});
```

### Custom Sources (e.g., LDAP)

Implement the `Security\Auth\Interfaces\UserSourceInterface`. Sources are typically resolved via the configuration in `App/Config/auth.php`.

```php
'sources' => [
    'ldap' => [
        'driver' => App\Auth\Sources\LdapSource::class,
    ],
],
```

## Technical Reference

### System Topology

|Component|Responsibility|Relevant Files|
|:---|:---|:---|
|**Auth Manager**|Singleton factory that resolves and caches Guards.|`System/Security/Auth/AuthManager.php`|
|**Guards**|Manage request-level authentication state (Session/Token).|`System/Security/Auth/Guards/`|
|**User Sources**|Abstract the retrieval of users from storage.|`System/Security/Auth/Sources/`|
|**Auth Services**|Core-level logic for Web or API flows.|`System/Security/Auth/AuthService.php`|

### Multi-Auth Architecture

In complex applications, you often have distinct user actors with completely different data structures and behaviors. Multi-guard allows you to keep these clean and isolated.

#### Scenario: EdTech Plattform (LMS)

- **admin**: Manages the platform and billing (Table: `admin`)
- **staff**: Teachers and Instructors (Table: `instructor`)
- **learner**: Students taking courses (Table: `student`)

```php
// Route Context configuration
'path' => 'classroom/{*}',
'context' => [
    'guards' => ['staff', 'learner'] // Allows both teachers and students
]
```

#### Scenario: Marketplace (E-commerce)

- **customer**: Regular shoppers (Standard `user` table)
- **vendor**: Sellers managing inventory (Table: `vendor`)
- **support**: Internal support agents (Table: `support_agent`)

```php
// In a Controller
public function dashboard(): Response
{
    $guard = $this->request->getRouteContext('auth_guard');
    
    return match($guard) {
        'vendor' => $this->asView('vendor.dashboard'),
        'customer' => $this->asView('customer.profile'),
        default => $this->response->redirect(url('login')),
    };
}
```

### Technical Benefits

- **Zero Nulls**: No need for a single "Users" table with 50+ nullable columns.
- **Security Isolation**: Database-level isolation for sensitive roles (Admins).
- **Scale**: Add new actor types (e.g. `guest`, `api-client`) without refactoring existing logic.

### Architecture: The Guard/Source Pattern

Anchor decouples **State Management** from **Data Retrieval**:

- **Guard**: Responsible for identifying the user from the incoming request (via session cookie, bearer token, etc.).
- **UserSource**: Responsible for finding the user in the database and verifying that their password matches.

### Multi-Guard Support

Anchor supports multiple authentication guards within a single application. This is useful for separating `Admin`, `Staff`, and `Learner` logins.

```php
// Switch guards fluently
$this->auth->viaGuard('admin')->login($request);
```

### Advanced Multi-Auth Features

#### Context-Aware `auth()` Helper

The global `auth()` helper is "Smart." It automatically detects the active guard set by the middleware:

```php
// In a controller protected by 'admin' guard
$admin = auth()->user(); // Automatically uses 'admin' guard
```

#### Global Logout

You can log a user out of **all** session-based guards at once:

```php
auth()->logoutAll();
// Or via the service
$this->auth->logoutAll();
```

#### Guard-Specific Redirects

Customize where users are sent when a guard check fails by using the `login_route` context:

```php
'auth' => [
    'admin/*' => [
        'guards' => ['admin'],
        'login_route' => 'admin.login' // Redirects to /admin/login instead of /login
    ]
]
```

### Configuration (`App/Config/auth.php`)

```php
return [
    'guards' => [
        'web' => ['driver' => 'session', 'source' => 'user'],
        'admin' => ['driver' => 'session', 'source' => 'admin'],
        'learner' => ['driver' => 'session', 'source' => 'learner'],
        'api' => ['driver' => 'token', 'source' => 'user'],
    ],
    'sources' => [
        'user'    => ['driver' => 'database', 'model' => App\Models\User::class],
        'admin'   => ['driver' => 'database', 'model' => App\Models\Admin::class],
        'learner' => ['driver' => 'database', 'model' => App\Models\Learner::class],
    ],
];
```

### Security & Hardening

- **Session Rotation**: The `SessionGuard` automatically regenerates the session ID upon login/logout to prevent fixation attacks.
- **Timing Attack Resistance**: All password verification uses `password_verify()` for constant-time comparisons.
- **Argon2ID**: Default hashing is handled via the `enc()` helper using the Argon2ID algorithm.
- **Event System**: The framework dispatches events (`LoginEvent`, `LogoutEvent`, `LoginFailedEvent`) for all major authentication actions, allowing for easy auditing and automated security responses (like IP banning).

### Operational Best Practices

- **CSRF Protection**: Always use `VerifyCsrfToken` middleware for web routes.
- **Secure Headers**: Ensure your application serves appropriate security headers.
- **Hidden Fields**: Use the `$hidden` property in your `BaseModel` to prevent sensitive data like passwords from leaking in JSON responses.
- **Database-Backed Sessions**: The `SessionGuard` integrates with the `SessionManagerInterface` (implemented by `SessionService`) to store session tokens in the database. This allows for:
  - **Remote Revocation**: Log out a user from all devices by deleting their database sessions.
  - **Session Auditing**: Track IP addresses and browser info for active sessions.
  - **Multi-Device Support**: Manage multiple active sessions per user securely.
