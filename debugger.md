# Debugger

Anchor comes with a powerful integrated debugging tool based on PHP DebugBar. It collects and displays relevant debugging information in a floating bar at the bottom of your browser.

## Installation

Debugger is a **core package** that requires installation before use.

### Install the Package

```bash
php dock package:install Debugger --system
```

This will:

- Copy configuration files to `App/Config/`
- Register the service provider
- Register middleware for web routes

### Verify Installation

Check that the configuration file exists (if package provides one):

```bash
ls App/Config/debugger.php
```

The debugger will automatically activate when `APP_DEBUG=true` and `APP_ENV=local`.

## Features

The Debugger automatically collects the following information for every request:

- **Messages**: Custom log messages.
- **Timeline**: Execution time of various events.
- **Exceptions**: Uncaught exceptions with stack traces.
- **Views**: Rendered templates and view data.
- **Route**: Current route information and parameters.
- **Queries**: Executed database queries with duration and bindings.
- **Models**: Loaded models.
- **Request**: HTTP request input and headers.
- **Session**: Session data.
- **Config**: Loaded configuration values.

## Usage

### Simple Logging

You can use the [Log Facade](helpers/log.md) to log messages. All log messages are automatically pushed to the debug bar's Messages collector (when debug mode is enabled) in addition to being written to files.

```php
use Helpers\Log;

// Log to default file (also appears in debug bar)
Log::channel('anchor')->info('User logged in', ['id' => $user->id]);

// Log to specific channel (also appears in debug bar)
Log::channel('payments')->info('Payment processed');

// Different log levels appear with different colors in debug bar
Log::channel('app')->error('Database connection failed');   // Red
Log::channel('app')->warning('Disk space low');             // Yellow
Log::channel('app')->info('Task completed');                // Blue
```

### Dump and Die

Standard debugging functions are enhanced:

```php
// Dump variable to debug bar
dump($variable);

// Dump variable and stop execution
dd($variable);
```

### Accessing the Debugger

You can interact directly with the `Debugger` service to push custom messages or access the underlying DebugBar instance.

```php
use Debugger\Debugger;

// Get debugger instance
$debugger = Debugger::getInstance();

// Push a custom message to the Messages collector
$debugger->push('messages', 'This is a custom debug message', 'info');
$debugger->push('messages', 'Warning: disk space low', 'warning');

// Access the underlying PHP DebugBar instance
$debugbar = $debugger->getDebugBar();
$renderer = $debugger->renderer();
```

## Configuration

The Debugger is enabled by default in development environments (`APP_ENV=local` and `APP_DEBUG=true`).

You can configure detailed settings in `App/Config/app.php` or `App/Config/debugger.php` if available (custom configuration).

## Security

Ensure `APP_DEBUG` is set to `false` in production environments to prevent sensitive information leakage. By default, the Debugger will not render if `APP_DEBUG` is false.

## State Management

In long-running environments, the `DebuggerServiceProvider` implements the `TerminableInterface`. When the system terminates (e.g., after a queue job), it calls `Debugger::terminate()` which clears all collected data and resets the `DebugBar` instance to prevent memory leaks and data overlap between cycles.
