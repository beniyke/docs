# Events

The Event system allows you to decouple various aspects of your application by implementing a simple Observer pattern. It enables you to handle "side effects" (like sending emails, logging, or updating wallets) without cluttering your main business logic.

## Core Concepts

- **Events**: Simple DTOs (Data Transfer Objects) that hold data about what happened.
- **Listeners**: Classes that perform an action when a specific event is fired.
- **Dispatcher**: Managed by `Core\Event`, it handles the registration and execution of listeners.

## Basic Usage

### Defining Events

Events are plain PHP classes. They can reside anywhere in your codebase making the framework highly flexible.

- **package**: Core framework events (e.g., `Pay\Events\PaymentSuccessfulEvent`).
- **App**: Application-specific events (e.g., `App\Events\UserRegisteredEvent`).

**Example:** `App\Events\UserRegisteredEvent.php`

```php
namespace App\Events;

use App\Models\User;

class UserRegisteredEvent
{
    public function __construct(
        public User $user
    ) {}
}
```

### Defining Listeners

Listeners handle the event logic. They must implement a `handle` method which receives the event instance.

**Example:** `App\Shop\Listeners\CreateCustomerProfileListener.php`

```php
namespace App\Shop\Listeners;

use App\Events\UserRegisteredEvent;

class CreateCustomerProfileListener
{
    public function handle(UserRegisteredEvent $event): void
    {
        // Access event data
        $user = $event->user;

        // Logic to create a shop customer profile...
    }
}
```

### Registering Listeners

You register listeners (typically in a `ServiceProvider`) using the `Event::listen` method.

```php
use Core\Event;
use App\Events\UserRegisteredEvent;
use App\Shop\Listeners\CreateCustomerProfileListener;

public function boot(): void
{
    Event::listen(UserRegisteredEvent::class, CreateCustomerProfileListener::class);
}
```

### Dispatching Events

To fire an event, use the static `dispatch` method. This will synchronously execute all registered listeners (unless they are queued).

```php
use Core\Event;
use App\Events\UserRegisteredEvent;

// Assuming $user is a valid User model instance
Event::dispatch(new UserRegisteredEvent($user));
```

## Advanced Usage

### Dependency Injection

Listeners are resolved via the **Service Container**. You can type-hint dependencies (like Repositories, Loggers, or Services) in the listener's `__construct` method, and they will be automatically injected.

```php
use Helpers\File\Contracts\LoggerInterface;
use Wallet\Services\WalletManager;

public function __construct(
    private LoggerInterface $logger,
    private WalletManager $wallet
) {}
```

### Stopping Propagation

If a listener returns `false` from its `handle` method, the event will stop propagating to any subsequent listeners.

### Asynchronous Listeners (Queues)

For heavy tasks (e.g., sending emails), you can offload the listener to the Queue system by implementing the `Core\Contracts\ShouldQueue` interface.

```php
use Core\Contracts\ShouldQueue;

class SendWelcomeEmailListener implements ShouldQueue
{
    public function handle(UserRegisteredEvent $event): void
    {
        // This will be executed by the Queue Worker automatically
    }
}
```

## Hybrid Structure Support

The Event system is designed to work across all layers of the Anchor Framework:

- **System**: Core events and listeners (e.g., Payment processing).
- **App**: Application-specific business logic and feature modules (e.g., `App\Shop`).

You can freely mix and match: specific App modules can listen to System events, or the System layer (if extended) could listen to App events.

## Robustness

- **Exception Isolation**: If a listener throws an exception, it is caught and logged, preventing the main request from failing.
- **Class Validation**: The dispatcher checks if listener classes exist and embody a `handle` method before execution.

## CLI Commands

### Generating Components

The CLI automatically ensures that **Event** and **Listener** suffixes are applied to your class names.

```bash
# Suffixes are automatically appended if missing
php dock event:create OrderPlaced      # Generates OrderPlacedEvent.php
php dock listener:create SendInvoices  # Generates SendInvoicesListener.php
```

### Reference

```bash
# Create an event
php dock event:create UserRegistered

# Create an event in a module
php dock event:create OrderPlaced Shop

# Create a listener
php dock listener:create SendWelcomeEmail

# Create a listener in a module
php dock listener:create UpdateInventory Shop

# Delete events/listeners (Factors in suffixes too)
php dock event:delete UserRegistered
php dock listener:delete SendWelcomeEmail
```

See [CLI Documentation](cli.md) for full command reference.

## System Events

The framework provides several internal events that you can listen to for lower-level lifecycle hooks. These are managed by the internal `Core\Providers\EventServiceProvider`.

| Event | Description | Dispatched By |
| --- | --- | --- |
| `Core\Events\KernelTerminateEvent` | Dispatched after the HTTP response is sent. | `App` |
| `Core\Events\ConsoleTerminateEvent` | Dispatched after a CLI command finishes. | `Console` |

### System Listeners

Essential framework behaviors are implemented as system listeners. They reside in `System\Events\Listeners` and are registered automatically:

- **`ClearResourceCacheListener`**: Clears relevant query caches based on the request context.
- **`ProcessDeferredTasksListener`**: Executes tasks pushed via the `Defer` system.

## State Management

In long-running processes (like Queue Workers), the framework automatically resets the `Event` dispatcher using `Event::reset()` during the [Termination Phase](internals.md#the-termination-flow). This prevents dynamic event listeners from accumulating and causing memory leaks.
