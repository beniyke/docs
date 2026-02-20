# Kernel

The `Kernel` class is the heart of the application's bootstrapping process. It initializes the environment, loads configurations, and prepares the application for handling requests.

## Responsibilities

- **Container Initialization**: Sets up the IoC Container (`Core\Ioc\Container`).
- **Bootstrapping**: Runs the `Bootstrapper` to load environment variables (`.env`), helper functions, and autoloaders.
- **Error Handling**: Registers the global error handler (`Core\Services\ErrorHandler`).
- **Service Providers**: Registers and boots all configured service providers.
- **Configuration**: Sets application-wide settings like timezone.

## Usage

The Kernel is typically instantiated and booted in `System/Core/init.php`:

```php
use Core\Kernel;
use Core\Ioc\Container;

$container = Container::getInstance();
$kernel = new Kernel($container, $basePath);
$kernel->boot();
```

## Boot Process

- **Construct**: Initializes the Container.
- **Boot**:
  - Runs `Bootstrapper::run()`
  - Registers `ErrorHandler`
  - **Internal Provider Registration**: Registers core system providers (e.g., `EventServiceProvider`, `DatabaseServiceProvider`).
  - **App Provider Registration**: Registers providers from `App/Config/providers.php`.
  - Boots all registered providers.
  - Sets Timezone
