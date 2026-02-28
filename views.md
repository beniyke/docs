# Views

Views contain the HTML of your application and separate your controller/application logic from your presentation logic. Views are plain PHP files.

## View Location

Views are stored in module-specific directories:

```text
App/src/{Module}/Views/Templates/
```

For example:

- `App/src/Auth/Views/Templates/login.php`
- `App/src/Account/Views/Templates/profile.php`
- `App/src/Tweets/Views/Templates/index.php`

## Creating Views

Create a simple view file:

### Using CLI

```bash
# Create a view template 'index' in 'Tweets' module
php dock view:create-template index Tweets

# Create a view modal 'delete-confirm' in 'Account' module
php dock view:create-modal delete-confirm Account

# Delete a view template or modal
php dock view:delete-template index Tweets
php dock view:delete-modal delete-confirm Account
```

### Manual Creation

**`App/src/Tweets/Views/Templates/index.php`**

```php
<h1>Tweets</h1>

<ul>
    <?php foreach ($tweets as $tweet): ?>
        <li><?php echo $this->escape($tweet->message); ?></li>
    <?php endforeach; ?>
</ul>
```

## Rendering Views

In controllers, use `$this->asView()`:

```php
public function index()
{
    $tweets = Tweet::all();

    return $this->asView('index', compact('tweets'));
}
```

The view engine automatically looks in the current module's `Views/Templates/` directory.

> [!IMPORTANT]
> **Isolated Scope**: As of version 2.6.0, the view engine uses isolated variable scope during compilation. This prevents developer-defined variables from shadowing internal engine variables, ensuring a more robust rendering process. Any variables passed to the view are extracted within a closure, isolated from the `ViewEngine` instance's internal state.


### Dot Notation

You can use dot notation to traverse directories. This is equivalent to using forward slashes:

```php
// Loads App/src/Auth/Views/Templates/auth/login.php
return $this->asView('auth.login');

// Equivalent to:
return $this->asView('auth/login');
```

## Passing Data to Views

### Using Compact

```php
$name = 'John';
$email = 'john@example.com';

return $this->asView('profile', compact('name', 'email'));
```

### Using Array

```php
return $this->asView('profile', [
    'name' => 'John',
    'email' => 'john@example.com'
]);
```

## Template Inheritance

### Layouts

Layouts are stored in `App/Views/Templates/layouts/`:

**`App/Views/Templates/layouts/app.php`**

```php
<!DOCTYPE html>
<html>
<head>
    <title><?php echo $this->section('title'); ?></title>
    <link rel="stylesheet" href="<?php echo assets('css/app.css'); ?>">
</head>
<body>
    <nav>
        <!-- Navigation -->
    </nav>

    <main>
        <?php echo $this->section('content'); ?>
    </main>

    <footer>
        <!-- Footer -->
    </footer>
</body>
</html>
```

### Extending Layouts

**`App/src/Tweets/Views/Templates/index.php`**

```php
<?php echo $this->extend('app'); ?>

// Or use dot notation for nested layouts:

<?php echo $this->extend('layouts.app'); ?>


<?php $this->startSection('title'); ?>
    Tweets - My App
<?php $this->endSection(); ?>

<?php $this->startSection('content'); ?>
    <h1>All Tweets</h1>

    <?php foreach ($tweets as $tweet): ?>
        <div class="tweet">
            <?php echo $this->escape($tweet->message); ?>
        </div>
    <?php endforeach; ?>
<?php $this->endSection(); ?>
```

### Sections

Sections are placeholders for content in a layout.

```php
<!-- In your template -->
$this->startSection('title');
    echo 'Home Page';
$this->endSection();

<!-- In your layout -->
<title><?php echo $this->section('title', 'Default Title'); ?></title>
```

The `section()` method accepts an optional second argument for a default value if the section is not defined.

### Defining Sections

```php
<?php $this->startSection('sidebar'); ?>
    <div class="sidebar">
        <!-- Sidebar content -->
    </div>
<?php $this->endSection(); ?>
```

### Rendering Sections

In layout:

```php
<?php echo $this->section('sidebar'); ?>
```

### Checking if Section Exists

```php
<?php if ($this->hasSection('sidebar')): ?>
    <aside>
        <?php echo $this->section('sidebar'); ?>
    </aside>
<?php endif; ?>
```

## Including Partials

### Include Method

```php
<?php echo $this->include('partials/header', ['title' => 'My Page']); ?>

// Or dot notation:

<?php echo $this->include('partials.header', ['title' => 'My Page']); ?>

```

### Inc Method

for `inc/` folder

```php
<?php echo $this->inc('navigation', ['active' => 'home']); ?>

// Or:

<?php echo $this->inc('auth.nav'); ?>

```

This looks in `Views/Templates/inc/navigation.php`.

## View Methods

### asView

```php
asView(string $view, array $data = [], ?callable $callback = null): Response
```

Renders a view template from the current module's `Views/Templates` directory.

- **Example**:

```php
return $this->asView('profile', ['user' => $user], function() use ($user) {
    return !$user->is_active; // If true, renders default 'deny' template
});
```

#### Real World Use Cases for `asView` Callback

The `asView` callback is a powerful "last-mile" security check. It allows you to fetch your data first, and then decide based on that specific data if the user should actually see the page.

#### Scenario A: Resource Ownership Check (Boolean)

Use a boolean return to trigger the default `deny` template if the user doesn't own the resource being edited.

```php
public function edit(string $postId)
{
    $post = Post::find($postId);

    return $this->asView('edit-post', compact('post'), function() use ($post) {
        // If this returns TRUE, the view switches to the 'deny' template automatically.
        return $post->author_id !== $this->auth->user()->id;
    });
}
```

#### Scenario B: Feature Gating / Upselling (Array)

Use an array return to specify a custom template, such as an "Upgrade Required" page for premium features.

```php
public function analytics()
{
    $stats = $this->statsService->getDashboardData();

    return $this->asView('dashboard', compact('stats'), function() {
        if (! $this->auth->user()->isPro()) {
            // Return an array to specify exactly WHICH template to show instead.
            return [
                'value'    => true, 
                'template' => 'errors/upgrade-required' 
            ];
        }
        
        return false; // Access granted
    });
}
```

## Extensibility (Macros & Mixins)

The View Engine is `Macroable`, allowing you to add custom helper methods dynamically.

### Adding a Macro

```php
use Core\Views\ViewInterface;

// In a Service Provider boot method
container()->extend(ViewInterface::class, function($view) {
    $view::macro('uppercase', function($value) {
        return strtoupper($value);
    });
    return $view;
});
```

### Using a Mixin

Group multiple helpers into a single class:

```php
class StringMixins {
    public function bold() {
        return function($value) {
            return "<strong>{$value}</strong>";
        };
    }
}

$view->mixin(new StringMixins());
```

### section

```php
section(string $name): string
```

Renders the content of a defined section.

- **Use Case**: Used in layouts to placeholder where content from child views should be injected.

### extend

```php
extend(string $layout): void
```

Specifies which layout the current view should inherit from.

- **Example**: `<?php echo $this->extend('layouts.main'); ?>`.

### denyAccessIf

```php
denyAccessIf(bool $condition, string $template = 'deny'): self
```

Conditionally swaps the current template with another (usually an error or "access denied" template) if the condition is true.

- **Note**: While available in the view, it's most commonly used via the callback in `asView()`.
- **Example**:

```php
<?php $this->denyAccessIf(!$user->isAdmin(), 'errors.403'); ?>
```

### include

```php
include(string $partial, array $data = []): string
```

Includes a partial view from any location within the templates directory.

- **Example**: `<?php echo $this->include('partials/header', ['title' => 'Home']); ?>`.

### inc

```php
inc(string $file, array $data = []): string
```

A shortcut to include files specifically from the `inc/` subdirectory.

- **Example**: `<?php echo $this->inc('navbar'); ?>` loads `Views/Templates/inc/navbar.php`.

### modal

```php
modal(string $file, array $data = []): string
```

A shortcut to include files specifically from the `modals/` subdirectory.

- **Example**: `<?php echo $this->modal('delete-confirm'); ?>`.

### escape

```php
escape(string $value): string
```

Escapes HTML entities to prevent Cross-Site Scripting (XSS).

- **Security Note**: **Always** use this when echoing data provided by users.

### json

- **`json(mixed $data)`**: Safely encodes data to JSON for use in JavaScript.

```php
  <script>
      const config = <?php echo $this->json(['api' => '/api/v1']); ?>;
  </script>
```

### Navigation & Route Helpers

- **`active(string $path, string $class = 'active', string $default = '')`**: Returns a class string if the current URI matches the path.

  ```php
  <li <?php echo $this->active('/dashboard'); ?>>Dashboard</li>
  ```

- **`isRoute(string $name)`**: Checks if the current route matches the given name.

```php
  <?php if ($this->isRoute('login')): ?>
      <!-- Only shown on login page -->
  <?php endif; ?>
  ```

### Context & Environment Helpers

- **`user()`**: Access the currently authenticated user object.

```php
  Hello, <?php echo $this->user()->name; ?>
```

- **`session(?string $key = null, mixed $default = null)`**: Get session data.

```php
  <?php echo $this->session('flash_message'); ?>
```

- **`config(string $key, mixed $default = null)`**: Access configuration values.

```php
  Site Name: <?php echo $this->config('app.name'); ?>
```

- **`isLocal()`**: Returns `true` if the environment is `local`.
- **`isProduction()`**: Returns `true` if the environment is `production`.

```php
  <?php if ($this->isLocal()): ?>
      <script src="/dev-tool.js"></script>
  <?php endif; ?>
```

## Authentication Helpers

```php
<?php if ($this->auth()): ?>
    <p>Welcome, <?php echo $this->escape($user->name); ?>!</p>
<?php endif; ?>

<?php if ($this->guest()): ?>
    <a href="<?php echo url('login'); ?>">Login</a>
<?php endif; ?>
```

## Form & Validation Helpers

### Validation Errors

```php
<div class="form-group">
    <label>Email</label>
    <input type="email" name="email" <?php echo $this->class(['form-control', 'is-invalid' => $this->hasError('email')]); ?>>

    <?php if ($this->hasError('email')): ?>
        <span class="invalid-feedback"><?php echo $this->escape($this->error('email')); ?></span>
    <?php endif; ?>
</div>
```

### Old Input

```php
<input type="text" name="username" value="<?php echo $this->escape($this->old('username', $user->username)); ?>">
```

### Attribute Helpers

Cleanly render boolean attributes:

```php
<input type="checkbox" <?php echo $this->checked($user->is_active); ?>>
<option <?php echo $this->selected($value === $selected); ?>>...</option>
<button <?php echo $this->disabled($isProcessing); ?>>Submit</button>
<input type="text" <?php echo $this->readonly($isLocked); ?>>
<input type="email" <?php echo $this->required(true); ?>>
```

### Context & Route Helpers

Access route metadata and permissions directly within your views.

- **`context(string $key, mixed $default = null)`**: Get route context values (domain, entity, resource, action).
- **`canAccessAction(string $action)`**: Check if the user has permission for an action on the current resource (e.g., `edit`).
- **`isResourceActive(string $name, string $class = 'active', string $default = '')`**: Returns `$class` if the current resource matches `$name`.
- **`getRouteTitle(string $default)`**: Returns a title based on route permissions.
- **`getBreadcrumbs()`**: Returns an array of breadcrumbs for the current resource/action.

### Style Helper

Similarly for inline styles:

```php
<div <?php echo $this->style(['color: red' => $hasError, 'display: none' => ! $isVisible]); ?>></div>
```

## Stacks

Stacks allow you to push content to named locations, usually for scripts or styles in your layout.

### In Layout

```php
<head>
    <?php echo $this->stack('styles'); ?>
</head>
<body>
    ...
    <?php echo $this->stack('scripts'); ?>
</body>
```

### In View

```php
<?php $this->push('scripts'); ?>
    <script src="feature.js"></script>
<?php $this->endPush(); ?>

<?php $this->prepend('styles'); ?>
    <link rel="stylesheet" href="critical.css">
<?php $this->endPush(); ?>
```

## Once Rendering

Ensure a block of code is only rendered once per request, regardless of how many times it's called (useful for JS libraries):

```php
<?php if ($this->once('datepicker-js')): ?>
    <script src="datepicker.js"></script>
<?php endif; ?>
```

### CSRF Token

```php
<form method="POST">
    <?php echo $this->csrf(); ?>
    <!-- Form fields -->
</form>
```

### Method Spoofing

For PUT, PATCH, DELETE requests:

```php
<form method="POST">
    <?php echo $this->method('DELETE'); ?>
    <?php echo $this->csrf(); ?>
    <button type="submit">Delete</button>
</form>
```

### Important Form Fields

A convenient helper that generates **CSRF**, **Method Spoofing**, and **Callback** fields in one call:

```php
<form method="POST">
    <?php echo $this->importantFormFields('PUT'); ?>
    <!-- Generates: -->
    <!-- <input type="hidden" name="csrf_token" ... /> -->
    <!-- <input type="hidden" name="_method" value="PUT" /> -->
    <!-- <input type="hidden" name="callback_route" ... /> -->
</form>
```

### Hidden Fields

```php
<?php echo $this->hidden('user_id', $user->id); ?>
```

### Callback & Referer

Store routes for redirecting back after form submission:

```php
// Use current route as callback
<?php echo $this->callback(); ?>

// Use a specific route as callback
<?php echo $this->callback('account/profile'); ?>

// Use the referer (previous page) as callback
<?php echo $this->referer(); ?>
```

### Global Helper Functions

These are available everywhere:

#### HTML & Components

- **`html()`**: Returns the `HtmlBuilder` for fluent element generation.
- **`component(string $name)`**: Creates an `HtmlComponent` instance for reusable UI fragments.

#### Assets

```php
<link rel="stylesheet" href="<?php echo assets('css/app.css'); ?>">
<script src="<?php echo assets('js/app.js'); ?>"></script>
<img src="<?php echo assets('images/logo.png'); ?>">
```

### URL

```php
<a href="<?php echo url('account/profile'); ?>">Profile</a>
```

### Route Helpers

#### Route by Name

Get the route path from a named route (defined in `App/Config/route.php`):

```php
<?php echo route_name('home'); ?> // Returns: 'account/home'
```

To generate a full URL from a named route:

```php
<a href="<?php echo url(route_name('home')); ?>">Home</a>
```

#### Route Function

The `route()` function returns the current or specified route path:

```php
<?php echo route(); ?> // Current route path
<?php echo route('account/profile'); ?> // Specified route path
```

### Example

**Layout: `App/Views/Templates/layouts/master.php`**

```php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo $this->section('title') ?: 'My App'; ?></title>
    <link rel="stylesheet" href="<?php echo assets('css/app.css'); ?>">
    <?php echo $this->stack('styles'); ?>
</head>
<body>
    <header>
        <nav>
            <?php if ($this->auth()): ?>
                <a href="<?php echo url('account/profile'); ?>">Profile</a>
                <a href="<?php echo url('logout'); ?>">Logout</a>
            <?php else: ?>
                <a href="<?php echo url('login'); ?>">Login</a>
            <?php endif; ?>
        </nav>
    </header>

    <main class="container">
        <?php echo $this->section('content'); ?>
    </main>

    <?php echo $this->inc('footer'); ?>

    <script src="<?php echo assets('js/app.js'); ?>"></script>
    <?php echo $this->stack('scripts'); ?>
</body>
</html>
```

**View: `App/src/Account/Views/Templates/profile.php`**

```php
<?php echo $this->extend('master'); ?>

<?php $this->push('styles'); ?>
    <link rel="stylesheet" href="<?php echo assets('css/profile.css'); ?>">
<?php $this->endPush(); ?>

<?php $this->startSection('title'); ?>
    <?php echo $this->escape($user->name); ?> - Profile
<?php $this->endSection(); ?>

<?php $this->startSection('content'); ?>
    <h1>Profile</h1>

    <form method="POST" action="<?php echo url('account/update'); ?>">
        <?php echo $this->importantFormFields('PUT'); ?>

        <div class="form-group">
            <label>Name</label>
            <input type="text" name="name"
                <?php echo $this->class(['form-control', 'is-invalid' => $this->hasError('name')]); ?>
                value="<?php echo $this->escape($this->old('name', $user->name)); ?>">

            <?php if ($this->hasError('name')): ?>
                <span class="error"><?php echo $this->escape($this->error('name')); ?></span>
            <?php endif; ?>
        </div>

        <div class="form-group">
            <label>Email</label>
            <input type="email" name="email"
                <?php echo $this->class(['form-control', 'is-invalid' => $this->hasError('email')]); ?>
                value="<?php echo $this->escape($this->old('email', $user->email)); ?>">

            <?php if ($this->hasError('email')): ?>
                <span class="error"><?php echo $this->escape($this->error('email')); ?></span>
            <?php endif; ?>
        </div>

        <button type="submit">Update Profile</button>
    </form>
<?php $this->endSection(); ?>

<?php $this->push('scripts'); ?>
    <script src="<?php echo assets('js/profile.js'); ?>"></script>
<?php $this->endPush(); ?>
```

## Best Practices

- **Always escape output**: Use `$this->escape()` for user data
- **Use layouts**: Don't repeat HTML structure
- **Keep views simple**: Logic belongs in controllers/services
- **Use partials**: Break views into reusable components
- **Include CSRF tokens**: Always use `$this->csrf()` in forms
- **Use helper functions**: `assets()`, `url()`, `route()`
