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

// RESTful Resource routes (Index, Create, Store, Show, Edit, Update, Destroy)
Route::resource('photos', PhotoController::class);
```

### Fallback Routing

You can define a catch-all route that will be executed when no other manual or convention-based route matches:

```php
Route::fallback(function () {
    return view('errors.404');
});
```

### Route Grouping & Prefixes

Groups allow you to apply shared attributes like prefixes and middleware to multiple routes. Groups can be defined using attributes or a fluent API.

```php
// Using attributes array
Route::group(['prefix' => 'admin', 'middleware' => 'web'], function () {
    Route::get('dashboard', [AdminController::class, 'index']);
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
```

### Robust Middleware Support

Middleware can be assigned as a group name, a single class name, or an array.

```php
// Using a middleware group from config/middleware.php
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

- **Duplicate Names**: The **last registered** route for a given name wins. If `ModuleA` and `ModuleB` both define `->name('index')`, the one registered later (usually based on module loading order) will overwrite the first in the `route_url()` lookup.
- **Duplicate URIs**:
    - **Static Routes**: The last one registered for the same HTTP method and URI wins.
    - **Dynamic Routes**: The **first match** wins. If multiple dynamic patterns match the same URI, the router will execute the first one it finds in the registration order.

#### Best Practice: Namespacing

To avoid conflicts, follow these industry-standard patterns:

- **Prefix Route Names**: Always prefix your names with the module name (e.g., `blog.post.show` instead of `post.show`).
- **Use URI Groups**: Wrap your module's routes in a unique prefix to stay isolated from other modules:

```php
// In Blog/Route/map.php
Route::group(['prefix' => 'blog', 'as' => 'blog.'], function() {
    Route::get('posts', [PostController::class, 'index'])->name('index'); // Full name: blog.index, URI: /blog/posts
});
```

### Closure-Based Routes

Routes can be handled directly by a Closure with automatic dependency injection.

```php
use Helpers\Http\Response;
use App\Services\AnalyticsService;

Route::get('api/stats', function(AnalyticsService $service) {
    return new Response($service->getTodayStats());
});
```

#### Isolated View Rendering

When using `Route::view()`, templates are rendered in an isolated scope. This prevents **variable shadowing**, meaning you can safely pass variables like `data` or `view_template` without colliding with the engine's internal logic.

### Fallback Behavior

If a manual route is not matched, Anchor gracefully falls back to **Convention-Based Routing**. This ensures that existing auto-mapped routes continue to work alongside your manual definitions.

## Performance & Optimization

The routing system is designed for high performance:

- **O(1) Static Lookup**: Static routes use an internal hash map for near-instant resolution regardless of the number of routes.
- **Segment Pre-processing**: Route paths are pre-exploded during registration to minimize string operations during the request lifecycle.
- **Lazy Resolution**: Controller instantiation and dependency injection only occur after a definitive match is found.

## Advanced Configuration

Explicit configuration can still be managed in `App/Config/route.php` for:

- **Default Route**: Home page mapping.
- **Redirects**: Simple internal path aliases.
- **Named Routes**: Human-readable names for UI generation.

## CLI API Reference

| Command                         | Description                                           |
|---------------------------------|-------------------------------------------------------|
| `php dock route:setup {module}` | Generates `map.php` in the module's Route directory.  |
| `php dock route:delete {module}` | Deletes the `map.php` file from the module.           |
