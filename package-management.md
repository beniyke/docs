# Package Management

## Overview

The Anchor framework includes a powerful package management system that allows you to install, manage, and uninstall packages with ease. Packages can contain configurations, migrations, commands, service providers, and middleware.

> Not all packages require installation! The `package:install` and `package:uninstall` commands are **only needed** for packages that integrate with the application's core systems (service providers, middleware, migrations, or CLI commands). Standalone library packages that provide pure PHP classes can be used directly without installation.

## When Do You Need Package Installation?

**Install Required (Needs `setup.php`)**

Use `package:install` when your package needs to:

- Register **service providers** in `App/Config/providers.php`
- Add **middleware** to web or API routes
- Publish **configuration files** to `App/Config/`
- Run **database migrations**
- Provide **CLI commands** (automatically discovered)
  **Examples**: Authentication, Queue, Watcher, Payment Gateway, Multi-tenancy

### Installation NOT Required

Your package doesn't need installation if it:

- Only provides utility classes/helpers
- Is a pure PHP library (like a PDF generator, UUID library, etc.)
- Doesn't integrate with the application's configuration or service layer
- Is used via direct class instantiation (`new MyClass()` or `MyClass::method()`)

**Examples**: String utilities, Date formatters, Pure calculation libraries

## Downloading Packages

Before you can install a package that isn't already in your `packages/` directory, you need to download it from the official Anchor GitHub organization.

### The Download Command

Anchor provides a convenient `download` command to fetch packages via Git.

```bash
php dock download PackageName
```

> This command requires `git` to be installed and available in your system path.

### Options

| Option      | Shortcut | Description                                                       | Default |
| :---------- | :------- | :---------------------------------------------------------------- | :------ |
| `--branch`  | `-b`     | Specify a specific Git branch or tag to download                  | `main`  |
| `--install` | `-i`     | Automatically run the `package:install` process after downloading | `false` |

#### Examples

**Download a package:**

```bash
php dock download Ally
```

**Download and install in one step:**

```bash
php dock download Pay --install
```

**Download a specific version/branch:**

```bash
php dock download Bridge --branch v2.0
```

## Installation

Anchor provides a simple command to install packages. By default, the system will automatically search for the package in both the `packages/` and `System/` directories.

```bash
php dock package:install PackageName
```

### Explicit Source Flags

While auto-detection works in most cases, you can use these flags to be explicit or if you have name collisions:

| Flag | Description |
| :--- | :--- |
| `--packages` | Force installation from the `packages/` directory |
| `--system` | Force installation from the `System/` directory |

**Examples**:

```bash
# Auto-detect (Recommended)
php dock package:install Shield

# Explicit sources
php dock package:install Notifications --system
php dock package:install Watcher --packages
```

### What Happens During Installation?

- **Status Check**: The system verifies if the package is already installed
- **Self-Healing**: If partially installed (corrupted), prompts to repair
- **Publishing**:
  - **Recursively** copies configuration files from `Package/Config/` to `App/Config/`
  - **Recursively** copies migrations from `Package/Database/Migrations/` to `App/storage/database/migrations`
  - Preserves directory structure (subdirectories are copied intact)
  - Creates destination directories if they don't exist
- **Migration Execution**: **Automatically runs** package-specific migrations to create database tables
- **Registration**:
  - Adds service providers to `App/Config/providers.php`
  - Adds middleware to `App/Config/middleware.php`
- **Commands**: Package commands are auto-discovered (no copying needed)

> **Automatic Migration Execution**
> `package:install` **automatically runs** the package's migrations after publishing them. You don't need to manually run `php dock migration:run` unless you want to run additional pending migrations from other sources.

### Complete Installation Workflow

Since `package:install` handles both file publishing and migrations, a single command is usually all you need:

```bash
php dock package:install MyPackage
```

### Installation Statuses

| Status | Description |
| :--- | :--- |
| `INSTALLED` | All package files are present and registered |
| `MISSING_FILES` | Some files are missing (partial/corrupted installation) |
| `NOT_INSTALLED` | Package has never been installed |

### Auto-Repair Feature

If a package is in a `MISSING_FILES` state (e.g., you manually deleted a config file), the installer will detect this and prompt you to repair:

```
[WARNING] Package Watcher is in a partial state (some files missing).
Do you want to repair/re-install it? (yes/no) [yes]:
```

Selecting `yes` will:

- Uninstall the corrupted package completely
- Re-install it fresh

## Uninstallation

### Uninstalling a Package

```bash
php dock package:uninstall PackageName
```

**Example**:

```bash
php dock package:uninstall Watcher
```

### Non-Interactive Uninstallation

For automated scripts or CI/CD pipelines, use the `--yes` flag to skip the confirmation prompt:

```bash
php dock package:uninstall Watcher --packages --yes

```

### What Happens During Uninstallation?

- **Confirmation Prompt**: Asks for confirmation (destructive operation) - can be skipped with `--yes` flag
- **Migration Rollback**: Automatically runs `down()` methods for all package migrations (**drops tables**)
- **File Removal**:
  - Deletes configuration files from `App/Config/`
  - Deletes migration files from `App/storage/database/migrations/`
- **Unregistration**:
  - Removes service providers from `App/Config/providers.php`
  - Removes middleware from `App/Config/middleware.php`
- **Commands**: Package commands are removed automatically when uninstalled

> Uninstalling a package will **automatically delete data** by rolling back migrations. Always backup your database before uninstalling. You don't need to manually run `migration:rollback`.

## Creating Packages

### Minimum Package Structure (For Packages Needing Installation)

> This section applies only to packages that need installation. Standalone library packages don't need this structure - they can just be regular PHP classes.

A package that needs installation requires at minimum:

- A `setup.php` manifest file (required - triggers installation)
- At least one resource that needs integration:
  - Config file to publish to `App/Config/`
  - Migration to run
  - CLI command (auto-discovered from package)
  - Service provider to register
  - Middleware to add

**If your package doesn't have a `setup.php`, it doesn't use the installation system.**

#### Basic Package Example

```
System/MyPackage/
├── setup.php              # Manifest file (required)
├── Config/
│   └── mypackage.php     # Package configuration
├── Database/
│   └── Migrations/
│       └── 2025_01_01_000000_create_mypackage_table.php
├── Commands/
│   └── MyPackageCommand.php
└── Providers/
    └── MyPackageServiceProvider.php
```

### Creating a Complete Package

#### Step-by-Step

#### Step 1: Create the Directory Structure

```bash
mkdir -p System/MyPackage/{Config,Database/Migrations,Commands,Providers,Middleware}
```

#### Step 2: Create the Setup Manifest

Create `System/MyPackage/setup.php`:

```php
<?php

declare(strict_types=1);

return [
    'providers' => [
        MyPackage\Providers\MyPackageServiceProvider::class,
    ],
    'middleware' => [
        'web' => [
            MyPackage\Middleware\MyPackageMiddleware::class,
        ],
        'api' => [
            MyPackage\Middleware\ApiMiddleware::class,
        ],
    ],
];
```

#### Step 3: Create a Configuration File

Create `System/MyPackage/Config/mypackage.php`:

```php
<?php

declare(strict_types=1);

return [
    'enabled' => true,
    'default_timeout' => 30,
    'api_endpoint' => env('MYPACKAGE_API_ENDPOINT', 'https://api.example.com'),
    'features' => [
        'feature_one' => true,
        'feature_two' => false,
    ],
];
```

#### Step 4: Create a Migration

Create `System/MyPackage/Database/Migrations/2025_01_01_000000_create_mypackage_table.php`:

```php
<?php

declare(strict_types=1);

use Database\Migration\BaseMigration;
use Database\Schema\Schema;

class CreateMypackageTable extends BaseMigration
{
    public function up(): void
    {
        Schema::create('mypackage_items', function ($table) {
            $table->id();
            $table->string('name');
            $table->text('description')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('mypackage_items');
    }
}
```

#### Step 5: Create a Command

Create `System/MyPackage/Commands/MyPackageCommand.php`:

```php
<?php

declare(strict_types=1);

namespace MyPackage\Commands;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

class MyPackageCommand extends Command
{
    protected function configure(): void
    {
        $this->setName('mypackage:sync')
            ->setDescription('Synchronize MyPackage data');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);
        $io->success('MyPackage synchronized successfully!');

        return Command::SUCCESS;
    }
}
```

#### Step 6: Create a Service Provider

Create `System/MyPackage/Providers/MyPackageServiceProvider.php`:

```php
<?php

declare(strict_types=1);

namespace MyPackage\Providers;

use Core\Services\ServiceProvider;
use MyPackage\Services\MyPackageService;

class MyPackageServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->container->singleton(MyPackageService::class);
    }

    public function boot(): void
    {
        // Boot logic (if needed)
    }
}
```

#### Step 7: Install Your Package

```bash
php dock package:install MyPackage
```

This will automatically:

- Publish your configuration.
- Register your Service Provider and Middleware.
- **Run your migrations** to create the `mypackage_items` table.

#### Step 8: Verify Installation

You can now use your package immediately! Try running your sync command:

```bash
php dock mypackage:sync
```

## Package Structure

### Directory Structure Options

#### Minimal Package (Config Only)

```
System/SimplePackage/
├── setup.php
└── Config/
    └── simple.php
```

#### Standard Package (Config + Service Provider)

```
System/StandardPackage/
├── setup.php
├── Config/
│   └── standard.php
└── Providers/
    └── StandardServiceProvider.php
```

#### Full Package (All Features)

```
System/FullPackage/
├── setup.php
├── Config/
│   ├── config.php
│   └── api.php
├── Database/
│   └── Migrations/
│       ├── 2025_01_01_000000_create_table_one.php
│       └── 2025_01_02_000000_create_table_two.php
├── Commands/
│   ├── SyncCommand.php
│   └── CleanupCommand.php
├── Providers/
│   ├── ServiceProvider.php
│   └── EventServiceProvider.php
└── Middleware/
    ├── AuthMiddleware.php
    └── RateLimitMiddleware.php
```

### File Naming Conventions

| Type | Convention | Example |
| :--- | :--- | :--- |
| **Config** | `packagename.php` | `watcher.php`, `queue.php` |
| **Migration** | `YYYY_MM_DD_HHMMSS_description.php` | `2025_01_01_120000_create_users_table.php` |
| **Command** | `PascalCaseCommand.php` | `SyncDataCommand.php` |
| **Provider** | `PascalCaseServiceProvider.php` | `QueueServiceProvider.php` |
| **Middleware** | `PascalCaseMiddleware.php` | `AuthenticationMiddleware.php` |

## Setup Manifest Reference

The `setup.php` file is the package's manifest that tells the installer what to register.

### Structure

```php
<?php

declare(strict_types=1);

return [
    'providers' => [
        // List of service provider class names
    ],
    'middleware' => [
        'web' => [
            // Web middleware classes
        ],
        'api' => [
            // API middleware classes
        ],
    ],
];
```

### Examples

#### Package with Only Providers

```php
<?php

return [
    'providers' => [
        MyPackage\Providers\MyServiceProvider::class,
        MyPackage\Providers\MyEventServiceProvider::class,
    ],
    'middleware' => [],
];
```

#### Package with Only Middleware

```php
<?php

return [
    'providers' => [],
    'middleware' => [
        'web' => [
            MyPackage\Middleware\WebGuardMiddleware::class,
        ],
    ],
];
```

#### Package with Both (Common Pattern)

```php
<?php

return [
    'providers' => [
        Watcher\Providers\WatcherServiceProvider::class,
    ],
    'middleware' => [
        'web' => [
            Watcher\Middleware\WatcherMiddleware::class,
        ],
    ],
];
```

#### API-Focused Package

```php
<?php

return [
    'providers' => [
        ApiAuth\Providers\ApiServiceProvider::class,
    ],
    'middleware' => [
        'api' => [
            ApiAuth\Middleware\ApiKeyMiddleware::class,
            ApiAuth\Middleware\RateLimitMiddleware::class,
        ],
    ],
];
```

#### Empty Setup (Config/Migrations Only Package)

If your package only provides configs or migrations without any services:

```php
<?php

return [
    'providers' => [],
    'middleware' => [],
];
```

## Common Use Cases

### Analytics Package

Track user events and analytics

#### Structure

```
System/Analytics/
├── setup.php
├── Config/
│   └── analytics.php
├── Database/
│   └── Migrations/
│       └── 2025_01_15_000000_create_analytics_events_table.php
├── Providers/
│   └── AnalyticsServiceProvider.php
└── Middleware/
    └── TrackAnalyticsMiddleware.php
```

#### setup.php

```php
<?php

return [
    'providers' => [
        Analytics\Providers\AnalyticsServiceProvider::class,
    ],
    'middleware' => [
        'web' => [
            Analytics\Middleware\TrackAnalyticsMiddleware::class,
        ],
    ],
];
```

#### Installation

```bash
php dock package:install Analytics
```

### Payment Gateway Package

Integrate Stripe payment processing

#### Structure

```
System/PaymentGateway/
├── setup.php
├── Config/
│   └── payment.php
├── Database/
│   └── Migrations/
│       ├── 2025_01_20_000000_create_payments_table.php
│       └── 2025_01_20_000001_create_subscriptions_table.php
├── Commands/
│   ├── SyncPaymentsCommand.php
│   └── ProcessRefundsCommand.php
└── Providers/
    └── PaymentServiceProvider.php
```

#### setup.php

```php
<?php

return [
    'providers' => [
        PaymentGateway\Providers\PaymentServiceProvider::class,
    ],
    'middleware' => [],
];
```

#### Config (`payment.php`)

```php
<?php

return [
    'stripe' => [
        'public_key' => env('STRIPE_PUBLIC_KEY'),
        'secret_key' => env('STRIPE_SECRET_KEY'),
        'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
    ],
    'currency' => env('PAYMENT_CURRENCY', 'USD'),
    'supports_subscriptions' => true,
];
```

#### Installation

```bash
php dock package:install PaymentGateway
```

#### Available Commands After Install

```bash
php dock payments:sync
php dock payments:process-refunds
```

### Custom Authentication Package

Custom multi-factor authentication

#### Structure

```
packages/MFAAuth/
├── setup.php
├── Config/
│   └── mfa.php
├── Database/
│   └── Migrations/
│       └── 2025_01_25_000000_create_mfa_tokens_table.php
├── Middleware/
│   ├── RequireMFAMiddleware.php
│   └── ValidateMFATokenMiddleware.php
└── Providers/
    ├── MFAServiceProvider.php
    └── MFAEventServiceProvider.php
```

#### setup.php

```php
<?php

return [
    'providers' => [
        MFAAuth\Providers\MFAServiceProvider::class,
        MFAAuth\Providers\MFAEventServiceProvider::class,
    ],
    'middleware' => [
        'web' => [
            MFAAuth\Middleware\RequireMFAMiddleware::class,
        ],
        'api' => [
            MFAAuth\Middleware\ValidateMFATokenMiddleware::class,
        ],
    ],
];
```

#### Installation

```bash
php dock package:install MFAAuth
```

### Email Marketing Package

SaaS email marketing automation

#### Structure

```
packages/EmailMarketing/
├── setup.php
├── Config/
│   └── email_marketing.php
├── Database/
│   └── Migrations/
│       ├── 2025_02_01_000000_create_campaigns_table.php
│       ├── 2025_02_01_000001_create_subscribers_table.php
│       └── 2025_02_01_000002_create_campaign_logs_table.php
├── Commands/
│   ├── SendCampaignCommand.php
│   └── GenerateReportCommand.php
├── Providers/
│   └── EmailMarketingServiceProvider.php
└── Middleware/
    └── TrackEmailOpenMiddleware.php
```

#### setup.php

```php
<?php

return [
    'providers' => [
        EmailMarketing\Providers\EmailMarketingServiceProvider::class
        ],
    'middleware' => [
        'web' => [
            EmailMarketing\Middleware\TrackEmailOpenMiddleware::class,
        ],
    ],
];
```

#### Installation & Usage

```bash
php dock package:install EmailMarketing
php dock campaign:send --campaign=newsletter
```

### File Storage Package

Multi-driver cloud storage (no migrations needed)

#### Structure

```
System/CloudStorage/
├── setup.php
├── Config/
│   └── filesystems.php
└── Providers/
    └── StorageServiceProvider.php
```

#### Installation

```bash
php dock package:install CloudStorage
```

#### Usage

```php
$storage = resolve(StorageManager::class);
$storage->disk('s3')->put('file.pdf', $contents);
```

### Standalone Utility Package (NO Installation Required)

PDF generation library

#### Structure

```
System/PdfGenerator/
├── PdfGenerator.php
├── PdfTemplate.php
└── Helpers/
    ├── FontHelper.php
    └── ImageHelper.php
```

**NO `setup.php` file** - This package doesn't need installation!

**Usage** (Direct instantiation):

```php
use PdfGenerator\PdfGenerator;

$pdf = new PdfGenerator();
$pdf->addPage()
    ->addText('Hello World')
    ->save('output.pdf');
```

#### Why no installation needed?

- No service providers to register
- No middleware to add
- No migrations required
- No CLI commands
- Just pure utility classes used directly in your code

### Another Example - String Utilities

```
System/StringUtils/
├── StringHelper.php
├── UrlBuilder.php
└── Slugify.php
```

#### Usage

```php
use StringUtils\Slugify;

$slug = Slugify::make('Hello World'); // "hello-world"
```

These packages work like any standard PHP library - just use the classes directly!

## Best Practices

#### Namespace Your Package

Always use a unique namespace for your package to avoid conflicts:

```php
namespace MyCompany\MyPackage\Providers;
```

#### Use Environment Variables for Secrets

Never hardcode API keys or secrets in config files:

```php
// Bad
'api_key' => '12345-secret-key',

// Good
'api_key' => env('MYPACKAGE_API_KEY'),
```

#### Always Implement Migration `down()` Methods

Ensure migrations can be rolled back during uninstallation:

```php
public function down(): void
{
    Schema::dropIfExists('my_table');
}
```

#### Version Your Migrations

Use timestamps in migration filenames to avoid conflicts:

```
2025_01_15_120000_create_users_table.php
2025_01_15_120100_add_email_to_users.php
```

### Test Before Distribution

Always test the full installation/uninstallation cycle:

```bash
# Test install
php dock package:install MyPackage

# Test commands work
php dock mypackage:test

# Test uninstall
php dock package:uninstall MyPackage
```

## Troubleshooting

### Package Not Found

**Error**: `Package not found at: /path/to/package`

**Solution**: Verify the package directory exists. The system searches both `packages/` and `System/` automatically.

### Middleware Not Registered

**Issue**: Middleware not appearing in routes.

**Solution**: Check that `setup.php` has the correct middleware group (`web` or `api`).

### Command Not Available After Install

**Issue**: Command doesn't appear in `php dock` list.

**Solution**:

- Verify the command file exists in package's `Commands/` directory
- Ensure the command extends `Symfony\Component\Console\Command\Command`
- Check class namespace matches the directory structure
- Clear any PHP opcache: `php dock cache:clear` (if available)

### Partial Installation

**Issue**: Some files installed, others missing.

**Solution**: Run the installer again. It will auto-detect and prompt for repair:

```bash
php dock package:install MyPackage
```

### File Copy Failures

**Error**: `Failed to copy file: /source/path to /dest/path`

**Causes**:

- Insufficient permissions on destination directory
- Disk space full
- Source file doesn't exist or is locked

**Solution**:

- Check filesystem permissions: `chmod 755 App/Config`
- Verify disk space: `df -h`
- Ensure source files exist in the package directory
- Try running with elevated permissions (if safe)

## Summary

- Use `package:install` to install packages
- Use `package:uninstall` to cleanly remove packages
- Create `setup.php` as your package manifest
- Follow directory structure conventions
- Test your package before distribution
- The system auto-detects and repairs corrupted installations

For more information, see the [Service Providers](providers.md) and [Middleware](middleware.md) documentation.
