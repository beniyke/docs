# Routing

Anchor uses a hybrid approach to routing: **Convention over Configuration** by default, with support for explicit configuration.

## Convention-Based Routing

The default routing mechanism maps URLs directly to your module structure.

- **Pattern**: `/{module}/{controller}/{method}/{params...}`
- **Default Method**: If `{method}` is omitted, it defaults to `index`.
- **Use Case**: Rapid application development where manual route definitions aren't required for every page.
- **Example**: `/auth/login` maps to `App\Auth\Controllers\LoginController::index()`.

## Configuration-Based Routing

While Anchor relies on conventions, you can configure specific routing behaviors in `App/Config/route.php`.

### Default Route

Define the route to load when the root URL is accessed.

```php
'default' => 'website/home',
```

### Redirects

```php
'redirect' => [
    'login' => 'auth/login',
    'signup' => 'auth/signup',
],
```

Maps specific alias URLs to internal application paths.

- **Use Case**: Creating user-friendly URLs like `/login` instead of `/auth/login/index`.
- **Note**: These are internal redirects handled by the router.

### Substitutions

Substitute the first segment of a URL with another module name.

```php
'substitute' => [
    'admin' => 'backend',
],
```

### Named Routes

```php
'names' => [
    'dashboard' => 'account/home',
    'profile' => 'account/profile',
],
```

Assigns a readable name to a module/path combination.

- **Use Case**: Generating URLs in code without hardcoding paths. If the underlying path changes, you only update it in the config.
- **Example**: `url()->routeByName('dashboard')`.

## Middleware Configuration

Middleware is applied based on route groups defined in `App/Config/route.php`.

- **Wildcards**: Use `{*}` to match any character sequence (e.g., `account/{*}` matches all sub-paths under account).
- **Use Case**: Protecting an entire section of the site (like `/admin`) with a single filter rule.

```php
'auth' => [
    'web' => ['auth/{*}', 'account/{*}'],
    'api' => ['api/{*}'],
],
```
