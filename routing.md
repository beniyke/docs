# Routing

Anchor uses a hybrid approach to routing: **Convention over Configuration** by default, with support for powerful **Manual Routing** for complex use cases.

## Convention-Based Routing

The default routing mechanism maps URLs directly to your module structure.

- **Pattern**: `/{module}/{controller}/{method}/{params...}`
- **Default Method**: If `{method}` is omitted, it defaults to `index`.
- **Example**: `/auth/login` maps to `App\Auth\Controllers\LoginController::index()`.

## Manual Routing

Manual routing allows you to define explicit routes with support for groupings, middleware, regex constraints, and closures.

### Setting Up Manual Routing

To enable manual routing for a module, use the CLI:

```bash
php dock route:setup {ModuleName}
```

This creates a `Route/map.php` file in your module. To remove it:

```bash
php dock route:delete {ModuleName}
```

### Basic Route Definition

Manual routes are registered using the `Core\Route\Route` class.

```php
use Core\Route\Route;
use App\Account\Controllers\ProfileController;

// Simple GET route
Route::get('profile', [ProfileController::class, 'index']);

// Other HTTP methods
Route::post('profile/update', [ProfileController::class, 'update']);
Route::put('posts/{id}', [PostController::class, 'save']);
Route::patch('settings', [SettingsController::class, 'patch']);
Route::delete('user/{id}', [UserController::class, 'destroy']);
```

### Dynamic Parameters & Regex Constraints

You can use wildcards in your paths and enforce patterns using regex.

```php
// Dynamic parameter {id}
Route::get('user/{id}', [UserController::class, 'show'])
    ->where(['id' => '[0-9]+']);

// Optional parameters {id?}
Route::get('blog/{category?}', function($category = 'all') {
    return "Category: $category";
});

// Multiple parameters
Route::get('post/{slug}/{id}', [PostController::class, 'show'])
    ->where(['id' => '[0-9]+', 'slug' => '[a-z-]+']);
```

### Convenience Methods

Anchor provides several convenience methods for common routing patterns:

```php
// Match all common HTTP methods
Route::any('all-methods', [Controller::class, 'handle']);

// Match specific subset of methods
Route::match(['GET', 'POST'], 'submit', [Controller::class, 'submit']);

// Simple HTTP redirects
Route::redirect('old-path', 'new-path', 301);

// Direct view rendering (without a controller)
Route::view('welcome', 'pages.welcome', ['title' => 'Home']);

// Module View: Render a template from a specific module
Route::view('login', 'Auth::login');

// Absolute path: Render an arbitrary file directly
Route::view('test', '/absolute/path/to/template.php');

### RESTful Resource Routes

The `Route::resource` method registers all the necessary routes for a RESTful controller in a single line.

```php
Route::resource('photos', PhotoController::class);
```

This single declaration creates multiple routes to handle various actions on the resource:

| Verb      | URI                  | Action             | Route Name      |
|-----------|----------------------|--------------------|-----------------|
| GET       | `/photos`            | `index`            | `photos.index`  |
| GET       | `/photos/create`     | `create`           | `photos.create` |
| POST      | `/photos`            | `store`            | `photos.store`  |
| GET       | `/photos/{id}`       | `show`             | `photos.show`   |
| GET       | `/photos/{id}/edit`  | `edit`             | `photos.edit`   |
| PUT/PATCH | `/photos/{id}`       | `update`           | `photos.update` |
| DELETE    | `/photos/{id}`       | `destroy`          | `photos.destroy`|

### Fallback Routes

If you'd like to define a route that will be executed when no other route matches the incoming request, you may use the `Route::fallback` method. Typically, unhandled requests will automatically render a "404" page via your application's exception handler. However, since the fallback route is defined at the end of your routes, it allows you to define custom logic for missing pages.

```php
Route::fallback(function () {
    // Custom logic...
});
```

For cases where you just want to render a specific view for missing pages, you can use the more convenient `fallbackView` method:

```php
Route::fallbackView('errors.404');

// Fallback can also use module-based or absolute paths
Route::fallbackView('Blog::errors.404');
Route::fallbackView('/custom/views/error.php');
```

Just like `Route::view()`, the fallback version supports:

- **Module Paths**: Use `Module::template` to render from a specific module.
- **Absolute Paths**: Provide a full system path to bypass default template locations.
- **Isolated Scope**: Templates are rendered in an isolated closure (via `->call($this)`) to prevent variable shadowing while maintaining access to the `$this` context.

### Route Grouping & Prefixes

Groups allow you to apply shared attributes like prefixes and middleware to multiple routes. Groups can be defined using attributes or a fluent API.

```php
// Using attributes array
Route::group(['prefix' => 'admin', 'middleware' => 'web', 'as' => 'admin.'], function () {
    Route::get('dashboard', [AdminController::class, 'index'])->name('index'); // admin.index
});

// Using fluent API
Route::prefix('admin')->middleware('auth')->group(function() {
    Route::get('dashboard', [AdminController::class, 'index']);
});

// Chaining attributes without an explicit array
Route::middleware('auth')->group(function() {
    Route::get('profile', [ProfileController::class, 'show']);
});
```

### Route Names & URL Generation

Named routes allow you to decouple your application's logic from its URI structure.

```php
Route::get('user/profile/{username}', [AccountController::class, 'show'])
    ->name('profile.view');

// Generate URL: /user/profile/beniyke
$url = route_url('profile.view', ['username' => 'beniyke']);

// Generate URIs with query parameters
$url = route_url('profile.view', ['username' => 'beniyke', 'ref' => 'docs']);
// Result: /user/profile/beniyke?ref=docs
```

### Robust Middleware Support

Middleware can be assigned as a group name (defined in `config/middleware.php`), a single class name, or an array.

```php
// Using a middleware group
Route::middleware('web')->get('shop', [ShopController::class, 'index']);

// Chaining multiple middlewares
Route::middleware('auth')->middleware('verified')->group(function() {
    Route::get('billing', [BillingController::class, 'show']);
});
```

#### Middleware Inheritance & Overrides

Manual routes are **Secure by Default**. They automatically inherit global middleware defined in `App/Config/route.php` (e.g., `auth`) based on their URI patterns.

If you need to bypass these global defaults, Anchor provides fluent override methods:

```php
// Surgical Exclusion: Remove ONLY the 'auth' middleware
Route::get('account/public-info', [AccountController::class, 'info'])
    ->withoutMiddleware('auth');

// Total Override: Ignore ALL global defaults and use ONLY what is defined here
Route::get('account/exclusive', [AccountController::class, 'exclusive'])
    ->onlyMiddleware(['special_api_auth']);

// Fluent Entry for Exclusive Groups
Route::exclusive()->group(function() {
    Route::get('public-api', [ApiController::class, 'index']);
});
```

### Handling Route Conflicts

In a multi-module environment, it is possible for different modules to define routes with identical names or URIs. Anchor resolves these conflicts as follows:

- **Duplicate Names**: The **last registered** route for a given name wins.
- **Duplicate URIs**:
    - **Static Routes**: The last one registered for the same HTTP method and URI wins.
    - **Dynamic Routes**: The **first match** wins. If multiple dynamic patterns match the same URI, the router will execute the first one it finds in the registration order.

### Closure-Based Routes

Routes can be handled directly by a Closure with automatic dependency injection from the container.

```php
use Helpers\Http\Response;
use App\Services\AnalyticsService;

Route::get('api/stats', function(AnalyticsService $service) {
    return new Response($service->getTodayStats());
});
```

### Fallback Behavior

If a manual route is not matched, Anchor gracefully falls back to **Convention-Based Routing**. This ensures that existing auto-mapped routes continue to work alongside your manual definitions.

---

## Configuration-Based Routing

While `Route::map()` is the primary way to define manual routes, some routing features are managed directly in `App/Config/route.php`.

### Default Home Page

Assign the root URI `/` to a specific module and controller:

```php
'default' => 'website/home', // Maps / to App\Website\Controllers\HomeController::index()
```

### Static Redirects

Quickly alias paths to other URLs:

```php
'redirect' => [
    'login' => 'auth/login',
    'signup' => 'auth/signup',
],
```

### Route Substitutions (Aliasing)

Map a URL segment to a different module name. Useful for shortening URLs:

```php
'substitute' => [
    'me' => 'account', // /me/settings maps to the 'Account' module
],
```

### Global Named Routes

Register names for convention-based routes that don't have a manual map:

```php
'names' => [
    'home' => 'account/home',
],
```

---

## Auto-Security & Path Collections

Anchor provides a unique "Auto-Security" feature for convention-based routes. You can define middleware protection for entire path categories in `App/Config/route.php`.

### Path Collections Syntax

Anchor supports a powerful curly-brace syntax for compact path definitions:

- **Wildcards**: `auth/{*}` matches all sub-paths under `auth/`.
- **Collections**: `auth/{login, signup, reset}` matches exactly those three sub-paths.

### Example: Global Auth Enforcement

```php
// in App/Config/route.php
return [
    'auth' => [
        'web' => ['account/{*}', 'admin/{*}'],
    ],
    'auth-exclude' => [
        'web' => ['admin/login'], // Bypass auth for specifically /admin/login
    ]
];
```

---

## Route Context

Every resolved route is hydrated with "Context" metadata (Domain, Entity, Resource, Action). This powers smart features like automated validation, SEO titles, and audit logs.

- **Learn more**: [Route Context Documentation](route-context.md)

## Performance & Optimization

The routing system is designed for high performance:

- **O(1) Static Lookup**: Static routes use an internal hash map for near-instant resolution.
- **Segment Pre-processing**: Route paths are pre-exploded during registration to avoid redundant string operations.
- **Lazy Resolution**: Controller instantiation only occurs after a definitive match is found.

## CLI API Reference

| Command                         | Description                                           |
|---------------------------------|-------------------------------------------------------|
| `php dock route:setup {module}` | Generates `map.php` in the module's Route directory.  |
| `php dock route:delete {module}` | Deletes the `map.php` file from the module.           |
