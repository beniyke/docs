# Defer System

The `Defer` system allows you to postpone the execution of heavy tasks until after the response has been sent to the user. This improves perceived performance.

## Usage

Use the global `defer` helper function:

```php
use App\Notifications\Email\WelcomeEmail;
use Helpers\Data;

defer(function () {
    // Heavy task, e.g., sending email
    $payload = Data::make(['name' => 'John', 'email' => 'john@example.com']);
    resolve(Mail\Mailer::class)->send(new WelcomeEmail($payload));
});
```

## Named Scopes

You can push tasks to specific named scopes using the `name()` method on the deferrer:

```php
deferrer()->name('emails')->push(function () {
    // ...
});
```

## The Deferrer Service

You can also access the underlying `DeferrerInterface` directly:

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
- **Execution**: After the response is sent (using `fastcgi_finish_request()` or similar), the deferred callbacks are executed.

## Requirements

- **PHP-FPM**: For `fastcgi_finish_request()` to work effectively.
- **Container**: The `DeferrerInterface` must be bound in the container.
