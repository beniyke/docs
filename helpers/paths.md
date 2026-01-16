# Paths Helper

The `Helpers\File\Paths` class provides static methods to get absolute paths to common application directories. It ensures consistent path handling across all platforms.

> Use the `base_path()`, `app_path()`, `storage_path()`, `public_path()`, and `config_path()` global functions for convenient access.

```php
// Get storage path
$logPath = Paths::storagePath('logs/app.log');
// /var/www/myapp/App/storage/logs/app.log

// Using global helpers
$configPath = config_path('database.php');
```

#### setBasePath

```php
static setBasePath(string $path): void
```

Manually sets the application base path. Usually called during framework boot.

## Core Paths

### basePath

```php
static basePath(?string $value = null): string
```

Returns the absolute path to the project root directory.

```php
Paths::basePath();           // /var/www/myapp
Paths::basePath('vendor');   // /var/www/myapp/vendor
```

### appPath

```php
static appPath(?string $value = null): string
```

Returns the path to the `App` directory.

```php
Paths::appPath();              // /var/www/myapp/App
Paths::appPath('Models');      // /var/www/myapp/App/Models
```

### systemPath

```php
static systemPath(?string $value = null): string
```

Returns the path to the `System` directory (framework core).

```php
Paths::systemPath();           // /var/www/myapp/System
Paths::systemPath('Core');     // /var/www/myapp/System/Core
```

### appSourcePath

```php
static appSourcePath(?string $value = null): string
```

Returns the path to `App/src` (module source code).

```php
Paths::appSourcePath();           // /var/www/myapp/App/src
Paths::appSourcePath('Shop');     // /var/www/myapp/App/src/Shop
```

## Configuration & Storage

### configPath

```php
static configPath(?string $value = null): string
```

Returns the path to the configuration directory.

```php
Paths::configPath();                // /var/www/myapp/App/Config
Paths::configPath('database.php');  // /var/www/myapp/App/Config/database.php
```

### storagePath

```php
static storagePath(?string $value = null): string
```

Returns the path to the storage directory.

```php
Paths::storagePath();              // /var/www/myapp/App/storage
Paths::storagePath('logs');        // /var/www/myapp/App/storage/logs
Paths::storagePath('uploads/images'); // /var/www/myapp/App/storage/uploads/images
```

### cachePath

```php
static cachePath(?string $value = null): string
```

Returns the path to the cache directory.

```php
Paths::cachePath();           // /var/www/myapp/App/storage/cache
Paths::cachePath('views');    // /var/www/myapp/App/storage/cache/views
```

### publicPath

```php
static publicPath(?string $value = null): string
```

Returns the path to the public directory (web root).

```php
Paths::publicPath();             // /var/www/myapp/public
Paths::publicPath('assets/css'); // /var/www/myapp/public/assets/css
```

## View Paths

### viewPath

```php
static viewPath(?string $value = null, ?string $module = null): string
```

Returns the path to views. Supports module-specific paths.

```php
// App-level views
Paths::viewPath();                    // /var/www/myapp/App/Views
Paths::viewPath('home.php');          // /var/www/myapp/App/Views/home.php

// Module-specific views
Paths::viewPath('index.php', 'Shop'); // /var/www/myapp/App/src/Shop/Views/index.php
```

### templatePath

```php
static templatePath(?string $value = null, ?string $module = null): string
```

Returns the path to view templates.

```php
Paths::templatePath();                        // /var/www/myapp/App/Views/Templates
Paths::templatePath('header.php');            // /var/www/myapp/App/Views/Templates/header.php
Paths::templatePath('sidebar.php', 'Admin');  // /var/www/myapp/App/src/Admin/Views/Templates/sidebar.php
```

### layoutPath

```php
static layoutPath(?string $value = null, ?string $module = null): string
```

Returns the path to layout templates.

```php
Paths::layoutPath();                        // /var/www/myapp/App/Views/Templates/layouts
Paths::layoutPath('main.php');              // /var/www/myapp/App/Views/Templates/layouts/main.php
Paths::layoutPath('dashboard.php', 'Admin'); // /var/www/myapp/App/src/Admin/Views/Templates/layouts/dashboard.php
```

## Framework Paths

### corePath

```php
static corePath(?string $value = null): string
```

Returns the path to the framework core.

```php
Paths::corePath();            // /var/www/myapp/System/Core
Paths::corePath('Event.php'); // /var/www/myapp/System/Core/Event.php
```

### cliPath

```php
static cliPath(?string $value = null): string
```

Returns the path to CLI components.

```php
Paths::cliPath();                  // /var/www/myapp/System/Cli
Paths::cliPath('Commands');        // /var/www/myapp/System/Cli/Commands
```

### testPath

```php
static testPath(?string $value = null): string
```

Returns the path to the tests directory.

```php
Paths::testPath();                     // /var/www/myapp/tests
Paths::testPath('System/Unit');        // /var/www/myapp/tests/System/Unit
```

### coreViewPath

```php
static coreViewPath(?string $value = null): string
```

Returns the path to framework core views.

````php
Paths::coreViewPath();          // /var/www/myapp/System/Core/Views

#### coreViewTemplatePath

```php
static coreViewTemplatePath(?string $value = null): string
````

Returns the path to framework core view templates.

## Path Utilities

### join

```php
static join(string ...$paths): string
```

Joins path segments using the correct OS separator.

```php
$path = Paths::join('/var/www', 'myapp', 'storage', 'logs');
// /var/www/myapp/storage/logs
```

### normalize

```php
static normalize(string $path): string
```

Normalizes path separators for the current OS and removes duplicate separators.

```php
$path = Paths::normalize('/var/www//myapp\\storage/logs');
// /var/www/myapp/storage/logs (on Linux)
// \var\www\myapp\storage\logs (on Windows)
```

### basename

```php
static basename(string $path): string
```

Returns the trailing name component of a path.

```php
Paths::basename('/var/www/myapp/file.txt'); // file.txt
```

### dirname

```php
static dirname(string $path): string
```

Returns the parent directory of a path.

```php
Paths::dirname('/var/www/myapp/file.txt'); // /var/www/myapp
```

## Global Helper Functions

| Function              | Equivalent                  |
| :-------------------- | :-------------------------- |
| `base_path($path)`    | `Paths::basePath($path)`    |
| `app_path($path)`     | `Paths::appPath($path)`     |
| `storage_path($path)` | `Paths::storagePath($path)` |
| `public_path($path)`  | `Paths::publicPath($path)`  |
| `config_path($path)`  | `Paths::configPath($path)`  |

## Examples

### Dynamic File Storage

```php
function storeUserUpload(int $userId, UploadedFile $file): string
{
    $directory = Paths::storagePath("uploads/users/{$userId}");

    if (!FileSystem::exists($directory)) {
        FileSystem::mkdir($directory);
    }

    $filename = uniqid() . '.' . $file->getExtension();
    $destination = Paths::join($directory, $filename);

    $file->moveTo($destination);

    return $destination;
}
```

### Module-Aware View Loading

```php
function loadView(string $view, ?string $module = null): string
{
    $path = Paths::viewPath($view . '.php', $module);

    if (!FileSystem::exists($path)) {
        throw new ViewNotFoundException("View not found: {$path}");
    }

    return $path;
}
```

### Cross-Platform Path Building

```php
function getLogPath(string $date): string
{
    return Paths::join(
        Paths::storagePath('logs'),
        date('Y', strtotime($date)),
        date('m', strtotime($date)),
        date('d', strtotime($date)) . '.log'
    );
}
```

## Related

- [FileSystem](filesystem-helper.md) - File operations
- [Cache](cache-helper.md) - File-based caching
