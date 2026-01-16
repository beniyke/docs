# Best Practices

This guide outlines the coding standards, patterns, and best practices for the Anchor framework. Following these practices ensures code quality, maintainability, and consistency across the codebase.

## Dependency Injection

The framework uses **constructor injection** as the primary dependency injection pattern. This approach improves testability, reduces coupling, and makes dependencies explicit.

### DO: Use Constructor Injection

```php
<?php

namespace MyPackage\Services;

use Core\Services\ConfigServiceInterface;
use Helpers\File\Adapters\Interfaces\FileMetaInterface;

class MyService
{
    public function __construct(
        private ConfigServiceInterface $config,
        private FileMetaInterface $fileMeta
    ) {}

    public function doSomething(): void
    {
        $path = $this->config->get('mypackage.path');
        if ($this->fileMeta->exists($path)) {
            // Process file
        }
    }
}
```

**Why?**

- Dependencies are explicit and visible
- Easy to mock in tests
- Impossible to create instance without required dependencies

### Dependency Injection vs. Global Helpers

The Anchor framework provides two primary ways to access system services: **Constructor Injection** and **Global Helper Functions**. Choosing the right one depends on your context and requirements.

#### When to use Constructor Injection (Recommended for Core Logic)

For complex services, package development, or components that require deep unit testing, constructor injection is the gold standard.

```php
// Highly testable, explicit dependencies
class ProfileService
{
    public function __construct(
        private ConfigServiceInterface $config,
        private SessionInterface $session
    ) {}

    public function getTheme(): string
    {
        return $this->session->get('theme') ?? $this->config->get('app.default_theme');
    }
}
```

#### When to use Global Helpers (Best for DX & Controllers)

Global helpers like `config()`, `session()`, `request()`, and `response()` are designed for rapid development. They are perfectly acceptable in Controllers, Views, and non-critical application logic where brevity improves readability.

```php
// Clean, concise, and fast to write
class ProfileController
{
    public function update()
    {
        $theme = request()->get('theme');
        session()->set('theme', $theme);

        return redirect()->back();
    }
}
```

> **The Golden Rule**: Use Constructor Injection if you need to mock the dependency in a unit test. Use Global Helpers if the convenience outweighs the need for strict isolation.

### Interface Binding Patterns

#### Singleton Binding

Use when you need the same instance throughout the application lifecycle:

```php
<?php

namespace MyPackage\Providers;

use Core\Services\ServiceProvider;
use MyPackage\Services\CacheService;

class MyServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Same instance returned every time
        $this->container->singleton(CacheService::class);
    }
}
```

**Use cases**: Database connections, caching, configuration, logging

#### Transient Binding

Use when you need a fresh instance every time:

```php
<?php

public function register(): void
{
    // New instance on every resolve
    $this->container->bind(PaymentProcessor::class);
}
```

**Use cases**: Request handlers, form validators, DTOs

## File System & Abstractions

While native PHP functions like `file_exists()` or `mkdir()` work, the framework encourages using its abstraction layers (like the `FileSystem` helper) for consistency and to simplify cross-platform path handling.

### Available Interfaces

| Interface | Purpose | Common Methods |
| | | - |
| `PathResolverInterface` | Resolve framework paths | `basePath()`, `appPath()`, `storagePath()`, `systemPath()` |
| `FileMetaInterface` | File metadata operations | `exists()`, `isDir()`, `isFile()`, `size()` |
| `FileReadWriteInterface` | Read/write operations | `get()`, `put()` |
| `FileManipulationInterface` | File/directory manipulation | `copy()`, `move()`, `delete()`, `mkdir()` |

### DO: Use Injected Abstractions

```php
<?php

namespace Package;

use Helpers\File\Adapters\Interfaces\FileMetaInterface;
use Helpers\File\Adapters\Interfaces\FileManipulationInterface;
use Helpers\File\Adapters\Interfaces\PathResolverInterface;
use RuntimeException;

class PackageManager
{
    public function __construct(
        private PathResolverInterface $paths,
        private FileMetaInterface $fileMeta,
        private FileManipulationInterface $fileManipulation
    ) {}

    public function publishConfig(string $packagePath): int
    {
        $source = $packagePath . DIRECTORY_SEPARATOR . 'Config';

        // Use injected interface, not is_dir()
        if (!$this->fileMeta->isDir($source)) {
            return 0;
        }

        $dest = $this->paths->appPath('Config');

        // Use abstraction for copying
        return $this->copyDirectoryContents($source, $dest);
    }

    private function copyDirectoryContents(string $source, string $dest): int
    {
        // Validate with abstraction
        if (!$this->fileMeta->isDir($source)) {
            throw new RuntimeException("Source directory does not exist: {$source}");
        }

        // Create directory with abstraction
        if (!$this->fileMeta->isDir($dest)) {
            $this->fileManipulation->mkdir($dest, 0755, true);
        }

        // FilesystemIterator is acceptable for iteration
        $items = new \FilesystemIterator($source, \FilesystemIterator::SKIP_DOTS);
        $count = 0;

        foreach ($items as $item) {
            $target = $dest . DIRECTORY_SEPARATOR . $item->getBasename();

            if ($item->isDir()) {
                $count += $this->copyDirectoryContents($item->getPathname(), $target);
            } else {
                // Use abstraction for file operations
                if (!$this->fileManipulation->copy($item->getPathname(), $target)) {
                    throw new RuntimeException("Failed to copy: {$item->getPathname()}");
                }
                $count++;
            }
        }

        return $count;
    }
}
```

### Choosing Between Framework Abstractions and Native Functions

While native PHP functions like `is_dir()` and `copy()` are familiar, using the framework's abstractions (like `FileSystem`) provides several advantages:

- **Testability**: Easy to mock in unit tests.
- **Cross-platform Consistency**: Handles path separators and directory permissions automatically.
- **Advanced Features**: Methods like `FileSystem::replace()` provide atomic writes that native PHP doesn't offer out of the box.

```php
// ✅ Recommended: Use FileSystem helper for I/O operations
if (!FileSystem::isDir($source)) {
    return 0;
}

// ✅ Acceptable: Use native functions for simple, non-I/O operations
$extension = pathinfo($filename, PATHINFO_EXTENSION);
```

**Why use abstractions for I/O?**

- **Mocking**: You can test how your code handles a full disk or an unreadable file without actually touching the disk.
- **Portability**: Your code will work correctly on Windows, Linux, and Cloud Storage if you use the abstraction layer.

### Exception: SPL Classes

It's acceptable to use SPL classes like `FilesystemIterator`, `DirectoryIterator`, and `SplFileInfo` for iteration since they provide better performance and features than `scandir()`:

```php
<?php

// ✅ Acceptable - SPL is fine for iteration
$items = new \FilesystemIterator($dir, \FilesystemIterator::SKIP_DOTS);

foreach ($items as $item) {
    if ($item->isDir()) {
        // Process directory
    }
}
```

## Error Handling

### When to Throw Exceptions

Throw exceptions for **exceptional conditions** that prevent normal execution:

```php
<?php

public function resolvePackagePath(string $package, bool $isSystem): string
{
    $base = $isSystem
        ? $this->paths->systemPath($package)
        : $this->paths->basePath("packages/{$package}");

    // Throw exception - caller cannot proceed without valid path
    if (!$this->fileMeta->isDir($base)) {
        throw new RuntimeException("Package not found at: {$base}");
    }

    return $base;
}
```

**Throw exceptions when:**

- Required resource doesn't exist (file, directory, database record)
- Invalid input that cannot be recovered
- System is in an invalid state
- External service fails

### When to Return Error Indicators

Return error codes/flags for **expected** alternative outcomes:

```php
<?php

public function publishConfig(string $packagePath): int
{
    $source = $packagePath . DIRECTORY_SEPARATOR . 'Config';

    // Return 0 - valid outcome, just nothing to do
    if (!$this->fileMeta->isDir($source)) {
        return 0;
    }

    return $this->copyDirectoryContents($source, $dest);
}
```

**Return error indicators when:**

- Optional operation has nothing to do (no files to copy)
- Multiple valid outcomes exist (success, skip, partial)
- Caller needs to make decisions based on the result

### Custom Exception Hierarchy

For domain-specific errors, create custom exceptions:

```php
<?php

namespace Package\Exceptions;

use RuntimeException;

class PackageNotFoundException extends RuntimeException
{
    public static function forPackage(string $name, string $path): self
    {
        return new self("Package '{$name}' not found at: {$path}");
    }
}

class PackageInstallationException extends RuntimeException
{
    public static function copyFailed(string $file, string $reason): self
    {
        return new self("Failed to copy '{$file}': {$reason}");
    }
}
```

Usage:

```php
<?php

if (!$this->fileMeta->isDir($base)) {
    throw PackageNotFoundException::forPackage($package, $base);
}
```

## Type Coverage with PHPStan

PHPStan is a static analysis tool that catches type errors without running your code. The framework is configured to run at **level 5**, which provides a good balance between strictness and practicality.

### Running PHPStan

```bash
# Analyze entire codebase
php vendor/bin/phpstan analyse

# Analyze specific directory
php vendor/bin/phpstan analyse System/Package

# Analyze specific file
php vendor/bin/phpstan analyse System/Package/PackageManager.php
```

### Understanding PHPStan Levels

```
Level 0 - Basic checks (undefined variables, unknown properties)
Level 1 - Unknown classes
Level 2 - Unknown methods called on $this
Level 3 - Return type checks
Level 4 - Check for dead code
Level 5 ⭐ - Framework default - Check argument types
Level 6 - Report missing typehints
Level 7 - Report partial union types
Level 8 - Check for nullable types
Level 9 - Maximum strictness - mixed types not allowed
```

**Current Configuration** (`phpstan.neon`):

```neon
parameters:
    level: 5
    bootstrapFiles:
        - System/Core/init.php
    paths:
        - App
        - System
        - index.php
    excludePaths:
        - %currentWorkingDirectory%/App/storage/*
```

### Common PHPStan Errors and Fixes

#### Error: "Undefined variable '$variable'"

```php
<?php

// ❌ Bad
public function process(): void
{
    if ($condition) {
        $result = $this->calculate();
    }
    echo $result; // Error: $result might not be defined
}

// ✅ Good
public function process(): void
{
    $result = null; // Initialize

    if ($condition) {
        $result = $this->calculate();
    }

    if ($result !== null) {
        echo $result;
    }
}
```

#### Error: "Parameter #1 expects string, string|null given"

```php
<?php

// ❌ Bad
public function getConfigPath(): ?string
{
    return $this->config->get('path'); // Might return null
}

public function process(): void
{
    $path = $this->getConfigPath();
    $this->fileMeta->exists($path); // Error: expects string, got string|null
}

// ✅ Good - Option 1: Guard clause
public function process(): void
{
    $path = $this->getConfigPath();

    if ($path === null) {
        throw new RuntimeException('Config path not set');
    }

    $this->fileMeta->exists($path); // Now PHPStan knows it's string
}

// ✅ Good - Option 2: Null coalescing
public function process(): void
{
    $path = $this->getConfigPath() ?? '/default/path';
    $this->fileMeta->exists($path);
}
```

#### Error: "Expected type 'int'. Found 'string'"

```php
<?php

// ❌ Bad - Type mismatch
public function multiply(int $factor): int
{
    return $this->multiply(RoundingMode::CEILING); // CEILING is a string
}

// ✅ Good - Use proper types
class RoundingMode
{
    public const CEILING = 1;
    public const FLOOR = 2;
    public const HALF_UP = 3;
}

public function multiply(int $roundingMode): int
{
    return $this->multiply(RoundingMode::CEILING);
}
```

### Type Hints Best Practices

Always use type hints for:

```php
<?php

// ✅ Method parameters and return types
public function parse(string $input): array
{
    return json_decode($input, true);
}

// ✅ Class properties (PHP 7.4+)
class MyClass
{
    private ConfigServiceInterface $config;

    private ?string $cache = null; // Nullable type

    /** @var array<string, mixed> */
    private array $options = []; // PHPDoc for complex types
}

// ✅ Closure parameters
$callback = function (User $user): void {
    echo $user->getName();
};
```

## Architecture Testing with Pest

Architecture tests ensure your codebase follows structural rules and conventions. They catch violations like improper dependencies, naming violations, or layer breaches.

### Setting Up Architecture Tests

Pest supports architecture testing out of the box. Create a dedicated file:

**File**: `tests/System/Architecture/ArchitectureTest.php`

```php
<?php

declare(strict_types=1);

describe('Architecture Rules', function () {

    // Rule: No native file functions in Package namespace
    arch('Package namespace does not use native file functions')
        ->expect('Package')
        ->not->toUse([
            'file_exists',
            'is_dir',
            'is_file',
            'scandir',
            'copy',
            'unlink',
            'mkdir',
            'rmdir',
            'file_get_contents',
            'file_put_contents'
        ]);

    // Rule: Package classes use proper abstractions
    arch('Package classes only depend on approved interfaces')
        ->expect('Package\\PackageManager')
        ->toOnlyUse([
            'Core\\Services\\ConfigServiceInterface',
            'Helpers\\File\\Adapters\\Interfaces\\*',
            'Database\\*',
            'RuntimeException',
            'Throwable',
            // PHP native classes are OK
            'FilesystemIterator',
            'DirectoryIterator'
        ]);

    // Rule: Service providers follow naming convention
    arch('Service providers follow naming convention')
        ->expect('*\\Providers\\*')
        ->toHaveSuffix('ServiceProvider')
        ->toExtend('Core\\Services\\ServiceProvider');

    // Rule: Commands follow naming convention
    arch('Console commands follow naming convention')
        ->expect('*\\Commands\\*')
        ->toHaveSuffix('Command')
        ->toExtend('Symfony\\Component\\Console\\Command\\Command');

    // Rule: Middleware follows naming convention
    arch('Middleware follows naming convention')
        ->expect('*\\Middleware\\*')
        ->toHaveSuffix('Middleware');

    // Rule: Strict types declared everywhere
    arch('All PHP files declare strict types')
        ->expect('App')
        ->toUseStrictTypes();

    arch('System uses strict types')
        ->expect('System')
        ->toUseStrictTypes();

    // Rule: No debug functions in production code
    arch('Production code does not use debug functions')
        ->expect('App')
        ->not->toUse(['dd', 'dump', 'var_dump', 'print_r']);

    arch('System code does not use debug functions')
        ->expect('System')
        ->not->toUse(['dd', 'dump', 'var_dump', 'print_r']);
});
```

### Running Architecture Tests

```bash
# Run all architecture tests
php vendor/bin/pest tests/System/Architecture/ArchitectureTest.php

# Run specific test
php vendor/bin/pest --filter="Package namespace does not use native file functions"
```

### Common Architecture Rules

#### Layer Dependencies

```php
<?php

// Controllers should only use services, not models directly
arch('Controllers use services, not models')
    ->expect('App\\Controllers')
    ->not->toUse('App\\Models');

// Models should not use controllers or middleware
arch('Models are independent')
    ->expect('App\\Models')
    ->not->toUse(['App\\Controllers', 'App\\Middleware']);
```

#### Naming Conventions

```php
<?php

arch('Test classes end with Test')
    ->expect('Tests')
    ->toHaveSuffix('Test');

arch('Interface classes end with Interface')
    ->expect('*\\Interfaces\\*')
    ->toHaveSuffix('Interface');
```

#### Dependency Direction

```php
<?php

// System should never depend on App
arch('System is independent from App')
    ->expect('System')
    ->not->toUse('App');

// Helpers should not depend on business logic
arch('Helpers are pure utilities')
    ->expect('Helpers')
    ->not->toUse(['App', 'Package']);
```

## End-to-End Testing

End-to-end (E2E) tests verify complete workflows across multiple components, often involving real database operations and file system interactions.

### Integration vs Unit Tests

| Type | Scope | Dependencies | Speed | Purpose |
| | - | - | | - |
| **Unit** | Single class/method | Mocked | Fast | Test logic in isolation |
| **Integration** | Multiple classes | Real or mixed | Medium | Test component interaction |
| **E2E** | Full workflow | Real | Slow | Test complete user scenarios |

### Database Testing Strategy

#### Use TestCase Helpers

The framework's `TestCase` class provides helpers for database testing:

```php
<?php

use Tests\TestCase;

beforeEach(function () {
    // Automatic database setup & transaction start
    $this->refreshDatabase();
});

it('creates a user record', function () {
    // Insert test data
    DB::table('user')->insert([
        'name' => 'John Doe',
        'email' => 'john@example.com',
        'password' => enc()->hashPassword('secret'),
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Assert database state
    $this->assertDatabaseHas('user', [
        'email' => 'john@example.com',
    ]);
});
```

#### Transaction Rollback Pattern

For tests that modify the database, use transactions:

```php
<?php

beforeEach(function () {
    Database\DB::beginTransaction();
});

afterEach(function () {
    Database\DB::rollBack();
});

it('processes payment and updates order', function () {
    $order = createTestOrder();

    $service = resolve(PaymentService::class);
    $service->processPayment($order->id, 100.00);

    $this->assertDatabaseHas('order', [
        'id' => $order->id,
        'status' => 'paid',
    ]);

    // Automatically rolled back after test
});
```

### File System Testing

For tests involving file operations, use temporary directories:

```php
<?php

beforeEach(function () {
    $this->testDir = sys_get_temp_dir() . '/anchor_test_' . uniqid();
    mkdir($this->testDir, 0755, true);
});

afterEach(function () {
    $this->deleteDirectory($this->testDir);
});

it('copies package config files', function () {
    // Arrange: Create test package structure
    $packagePath = $this->testDir . '/TestPackage';
    mkdir($packagePath . '/Config', 0755, true);
    file_put_contents($packagePath . '/Config/test.php', '<?php return [];');

    // Act: Copy files
    $manager = resolve(PackageManager::class);
    $count = $manager->publishConfig($packagePath);

    // Assert
    expect($count)->toBe(1);
    expect(file_exists($this->testDir . '/App/Config/test.php'))->toBeTrue();
});
```

### Browser Testing Pattern

For browser-based E2E tests, use the framework's browser helpers:

```php
<?php

it('completes user registration flow', function () {
    // Start browser session
    $this->browse(function ($browser) {
        $browser->visit('/register')
            ->type('name', 'Jane Doe')
            ->type('email', 'jane@example.com')
            ->type('password', 'secret123')
            ->type('password_confirmation', 'secret123')
            ->press('Register')
            ->assertPathIs('/dashboard')
            ->assertSee('Welcome, Jane');
    });

    // Verify database
    $this->assertDatabaseHas('user', [
        'email' => 'jane@example.com',
    ]);
});
```

### Full Workflow Example

```php
<?php

describe('Package Installation E2E', function () {

    beforeEach(function () {
        $this->testPackagePath = sys_get_temp_dir() . '/test_pkg_' . uniqid();
        mkdir($this->testPackagePath . '/Config', 0755, true);
        mkdir($this->testPackagePath . '/Database/Migrations', 0755, true);

        // Create test package files
        file_put_contents(
            $this->testPackagePath . '/setup.php',
            "<?php return ['providers' => ['Test\\\\Provider'], 'middleware' => []];"
        );

        file_put_contents(
            $this->testPackagePath . '/Config/test.php',
            "<?php return ['enabled' => true];"
        );
    });

    afterEach(function () {
        $this->deleteDirectory($this->testPackagePath);
    });

    it('completes full package installation workflow', function () {
        $manager = resolve(PackageManager::class);

        // Step 1: Publish configs
        $configCount = $manager->publishConfig($this->testPackagePath);
        expect($configCount)->toBeGreaterThan(0);

        // Step 2: Publish migrations
        $migrationCount = $manager->publishMigrations($this->testPackagePath);
        expect($migrationCount)->toBeGreaterThan(0);

        // Step 3: Register services
        $manifest = $manager->getManifest($this->testPackagePath);
        $manager->registerProviders($manifest['providers'] ?? []);

        // Step 4: Verify installation status
        $status = $manager->checkStatus($this->testPackagePath);
        expect($status)->toBe(PackageManager::STATUS_INSTALLED);

        // Step 5: Verify files exist
        expect($this->fileMeta->exists($this->paths->appPath('Config/test.php')))->toBeTrue();
    });
});
```

### Best Practices for E2E Tests

1. **Keep them focused**: Test one complete workflow per test
2. **Use realistic data**: Mirror production scenarios
3. **Clean up thoroughly**: Reset database and file system after each test
4. **Run selectively**: E2E tests are slow, run them in CI or before deployment
5. **Document complex setups**: Add comments explaining multi-step arrangements

## Performance & Scaling at Scale

Anchor doesn't collapse because it's slow; it collapses because convenience is allowed to replace discipline. To build products that survive real growth, follow these scaling rules:

#### 1. No Uncontrolled Data Fetching

If a query returns "everything," it's a bug. Large datasets must always be constrained, paginated, or streamed using the framework's available tools.

#### 2. Relationships Must Be Intentional

Lazy loading should never be "accidental" at scale. If relationships are accessed, they must be explicitly planned, reviewed, and eager-loaded where necessary to avoid N+1 issues.

#### 3. Indexing is Part of Feature Delivery

Any new filter, lookup, or foreign reference must ship with a database index. Indexing is not a follow-up task; it is part of the feature itself.

#### 4. Lightweight Request Cycles

Anything slow, repeatable, or failure-prone (e.g., sending emails, external API calls, heavy processing) must run outside the user request cycle. Queues are mandatory once traffic exists.

#### 5. Collections are Not a Data Engine

If logic can be pushed to the database layer (SQL), it should be. Moving large datasets into memory for heavy collection work is a red flag for scalability.

#### 6. Mandatory Pagination for Lists

No exceptions. If a screen or API endpoint lists data that can grow over time, it must implement pagination from day one.

#### 7. The Database is Never a Cache

Repeat reads require a caching strategy. Use the `Cache` helper to store expensive results. Database load should represent the source of truth, not a convenient workspace for temporary data.

#### 8. Performance is Measured, Not Assumed

Review query counts, execution time, and memory usage during development. Use the `Benchmark` helper to quantify performance before and after changes.

#### 9. Query/Builder Convenience is Audited

Be cautious with Accessors, Casts, Appended Attributes, and Global Scopes. While convenient, they are performance-sensitive and can hide expensive operations.

#### 10. Scaling Rules are Documented Early

Set performance boundaries and scaling expectations for your codebase before the team or traffic doubles. Written discipline prevents technical debt.

## Summary

Following these best practices ensures:

- **Testable code** through dependency injection
- **Consistent patterns** across the codebase
- **Type safety** with PHPStan static analysis
- **Structural integrity** with architecture tests
- **Reliability** through comprehensive E2E testing

Refer back to this guide whenever you're implementing new features or refactoring existing code.
