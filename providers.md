# Service Providers

Service Providers are the central place to configure and bootstrap services in the Anchor framework. They are responsible for **binding services into the IoC container** and performing any **bootstrapping logic** that should run after all providers have been registered.

## Types of Providers

| Type                      | Description                                                                                                                                               | When to use                                                         |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `ServiceProvider`         | Basic provider with `register()` and optional `boot()` methods. It is loaded on every request.                                                            | For services that are always needed.                                |
| `DeferredServiceProvider` | Extends `ServiceProvider` and defines a static `provides(): array` method. The framework loads it **only when one of the provided services is resolved**. | For heavy services or optional features that should be lazy‑loaded. |

---

## Lifecycle

- **Registration** - The framework first registers **Core System Providers** (defined internally in the `Kernel`), followed by **Application Providers** listed in `App/Config/providers.php`.
- **Boot** - After all providers have been registered, the framework calls `boot()` on each provider (if the method exists). This is the place for actions that depend on other services being available, such as eager‑loading relationships or event listeners.

## System vs Application Providers

Anchor maintains a clear separation between framework core logic and application logic:

- **System Providers**: Managed internally by the `Kernel`. These provide essential infrastructure like Events, Deferred Tasks, Databases, and Security. They are "non-negotiable" and ensure the framework functions even if application config is modified.
- **Application Providers**: Defined by you in `App/Config/providers.php`. These are used for your business logic, feature modules, and third-party integrations.

## Generating Providers

You can generate a new provider using the CLI:

```bash
php dock provider:create Payment
```

This creates a new file in `App/Providers/PaymentProvider.php`.

## Basic Usage

### AppServiceProvider

`AppServiceProvider` is a standard provider used for application-wide bootstrapping.

```php
<?php
namespace App\Providers;

use App\Services\UserService;
use App\Services\SessionService;
use Core\Services\ServiceProvider;
use Database\BaseModel;

class AppServiceProvider extends ServiceProvider
{
    private static array $singletons = [
        SessionService::class,
        UserService::class,
    ];

    public function register(): void
    {
        foreach (static::$singletons as $singleton) {
            $this->container->singleton($singleton);
        }
    }

    public function boot(): void
    {
        // Enable automatic eager‑loading of relationships for all models
        BaseModel::automaticallyEagerLoadRelationships();
    }
}
```

- **register()** binds `SessionService` and `UserService` as singletons.
- **boot()** runs after all providers have been registered, configuring the ORM.

### Deferred Usage

`AuthServiceProvider` demonstrates how to lazily load services only when requested.

```php
<?php
namespace App\Providers;

use App\Services\Auth\ApiAuthService;
use App\Services\Auth\Interfaces\AuthServiceInterface;
use App\Services\Auth\WebAuthService;
use Core\Services\DeferredServiceProvider;
use Helpers\Http\Request;

class AuthServiceProvider extends DeferredServiceProvider
{
    public static function provides(): array
    {
        return [AuthServiceInterface::class];
    }

    public function register(): void
    {
        $this->container->singleton(WebAuthService::class);
        $this->container->singleton(ApiAuthService::class);

        // Resolve the appropriate implementation based on the request type
        $this->container->singleton(AuthServiceInterface::class, function ($container) {
            $request = $container->get(Request::class);
            return $request->routeIsApi()
                ? $container->get(ApiAuthService::class)
                : $container->get(WebAuthService::class);
        });
    }
}
```

- **provides()** declares that this provider is responsible for `AuthServiceInterface`.
- The framework keeps this provider dormant until `AuthServiceInterface` is requested from the container.
- **register()** uses a closure to implement conditional logic (Web vs API auth).

## Registering Providers

All providers are listed in `App/Config/providers.php`:

```php
<?php
return [
    App\Providers\CacheServiceProvider::class,
    App\Providers\AppServiceProvider::class,
    App\Providers\ApiTokenValidatorServiceProvider::class,
    App\Providers\AuthServiceProvider::class,
    App\Providers\ApiGuardServiceProvider::class,
    App\Providers\EncryptionServiceProvider::class,
    App\Providers\NotificationServiceProvider::class,
];
```

The framework loads this array during bootstrapping.

---

## Best Practices

- **Keep `register()` lightweight** – only bind services. Heavy logic belongs in `boot()` or in the service itself.
- **Prefer interfaces** – bind an interface to a concrete class to allow easy swapping.
- **Use deferred providers** for services that are not needed on every request (e.g., mailer, notification system).
- **Avoid service locator anti‑pattern** – inject dependencies via constructor or method injection instead of pulling them from the container inside business logic.
- **Group related bindings** – create a dedicated provider for a feature (e.g., `NotificationServiceProvider`) to keep the codebase modular.

## Further Reading

- [IoC Container](container.md) – Overview of the IoC container and helper functions (`container()`, `resolve()`).
- [Configuration](configuration.md) – How the `providers.php` file fits into the overall configuration system.
- [Services](services.md) – Details on the core services provided by the framework.
