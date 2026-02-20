# Defer

The `Defer` system allows you to postpone the execution of heavy tasks until after the response has been sent to the user. This improves perceived performance.

## Usage

You can use the `Defer` facade or the global `defer()` helper function:

```php
use App\Notifications\Email\WelcomeEmail;
use Helpers\Data;
use System\Defer\Defer;
use Mail\Mail;

// Using the Facade
Defer::push(function () {
    $payload = Data::make(['name' => 'John', 'email' => 'john@example.com']);
    Mail::send(new WelcomeEmail($payload));
});

// Using the helper
defer(function () {
    // ...
});
```

## Named Scopes

You can push tasks to specific named scopes to organize your background work:

```php
use System\Defer\Defer;

// Using the Facade
Defer::name('emails')->push(function () {
    // ...
});

// Using the helper/service
deferrer()->name('emails')->push(function () {
    // ...
});
```

## The Deferrer Service

The `Defer` facade is a static interface to the `DeferrerInterface` service. You can also access the service directly:

```php
$deferrer = deferrer();

// Push to default queue
$deferrer->push(function() { ... });

// Push to named queue
$deferrer->name('images')->push(function() { ... });
```

## How It Works

- **Queueing**: The closure is added to a queue in memory.
- **Response**: The application finishes processing the request and sends the response to the browser.
- **Execution**: Deferred callbacks are executed automatically after the response is sent (via `KernelTerminateEvent`) or after a CLI command finishes (via `ConsoleTerminateEvent`). This is handled by the `ProcessDeferredTasksListener`.

## Requirements

- **PHP-FPM**: For `fastcgi_finish_request()` to work effectively.
- **Container**: The `DeferrerInterface` must be bound in the container.
