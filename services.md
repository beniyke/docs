# Core Services

Anchor comes with a set of core services that handle fundamental framework operations. These services are bound to the service container and can be injected into your controllers, commands, or other services.

## Accessing Services

You can access services in two ways:

- **Dependency Injection** (Recommended): Type-hint the service interface or class in your constructor.
- **Service Locator**: Use the `resolve()` helper.

```php
use Core\Services\ConfigServiceInterface;

class MyController
{
    public function __construct(
        private ConfigServiceInterface $config
    ) {}

    public function index()
    {
        $debug = $this->config->get('app.debug');
    }
}
```

---

## Config Service

| Type          | Namespace                              |
| :------------ | :------------------------------------- |
| **Class**     | `Core\Services\ConfigService`          |
| **Interface** | `Core\Services\ConfigServiceInterface` |

The Config Service is responsible for loading and accessing configuration values from the `App/Config` directory and `.env` file. It supports dot-notation for nested values and automatic caching in production.

**Methods**

`get(string $key, mixed $default = null): mixed`

Retrieve a configuration value using dot notation.

```php
// Get a value from App/Config/app.php
$appName = $config->get('app.name');

// Get a nested value
$dbHost = $config->get('database.connections.mysql.host');

// Get with default value
$timezone = $config->get('app.timezone', 'UTC');
```

`all(): array`

Retrieve all configuration values as an associative array.

```php
$allConfig = $config->all();
```

`isDebugEnabled(): bool`

Helper to check if the application is in debug mode.

```php
if ($config->isDebugEnabled()) {
    // Show detailed errors
}
```

## CLI Service

| Type          | Namespace                           |
| :------------ | :---------------------------------- |
| **Class**     | `Core\Services\CliService`          |
| **Interface** | `Core\Services\CliServiceInterface` |

The CLI Service parses command-line arguments and options passed to the application. It is primarily used within Console Commands.

**Methods**

`getCommandName(): ?string`

Returns the name of the command being executed (e.g., `migration:run`).

`getArguments(): array`

Returns all positional arguments passed to the command.

`getArgument(int $index, mixed $default = null): mixed`

Get a specific argument by its index (0-based).

```php
// php dock user:create john
$username = $cli->getArgument(0); // "john"
```

`getOptions(): array`

Returns all options (flags) passed to the command.

`hasOption(string $name): bool`

Check if a specific option was passed. Supports short (`-f`) and long (`--force`) syntax.

```php
// php dock migrate:up --force
if ($cli->hasOption('force')) {
    // Run without confirmation
}
```

`getOption(string $name, mixed $default = null): mixed`

Get the value of an option.

```php
// php dock serve --port=8080
$port = $cli->getOption('port', 8000); // 8080
```

`isCommand(string $name): bool`

Checks if the current command matches the given name.

```php
if ($cli->isCommand('migration:run')) {
    // ...
}
```

---

## Environment Service (Dotenv)

| Type          | Namespace                       |
| :------------ | :------------------------------ |
| **Class**     | `Core\Services\Dotenv`          |
| **Interface** | `Core\Services\DotenvInterface` |

The Environment Service manages the `.env` file. It loads environment variables into `$_ENV` and `$_SERVER` and provides methods to read/write these values programmatically.

**Methods**

`load(): void`

Loads the `.env` file. If in production, it attempts to load from the cache first.

`getValue(string $key, mixed $default = null): mixed`

Get a value from the environment variables. Automatically casts `true`, `false`, `null`, and `empty` strings to their PHP equivalents.

```php
$isDebug = $dotenv->getValue('APP_DEBUG'); // true (boolean)
```

`setValue(string $key, mixed $value): void`

Sets a value in the `.env` file and updates the current environment.

```php
$dotenv->setValue('APP_MAINTENANCE', 'true');
```

`generateAndSaveAppKey(): void`

Generates a new 32-byte `APP_KEY` and saves it to the `.env` file.

`cache(): void`

Manually triggers the caching of environment variables to `.env.cache`. This is typically handled automatically in production during `load()`, but can be called manually if needed (e.g., during deployment).

## Creating Custom Services

You can create your own services to encapsulate business logic.

### Create the Service Class

You can generate a service using the CLI:

```bash
# Create PaymentService in Account module
php dock service:create Payment Account
```

Or create it manually:

```php
namespace App\Services;

class PaymentService
{
    public function process(float $amount): bool
    {
        // Logic...
        return true;
    }
}
```

### Register in a Provider

Add the service to `App\Providers\AppServiceProvider` (or a custom provider).

```php
// App/Providers/AppServiceProvider.php
use App\Services\PaymentService;

public function register(): void
{
    // Bind as a singleton
    $this->container->singleton(PaymentService::class);
}
```

### Use the Service

```php
use App\Services\PaymentService;

class CheckoutController
{
    public function __construct(
        private PaymentService $payment
    ) {}

    public function pay()
    {
        $this->payment->process(100.00);
    }
}
```
