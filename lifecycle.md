# Request Lifecycle

Understanding the request lifecycle is crucial for mastering Anchor. It describes the path a request takes from entering your application to the final response.

## Entry Point

All requests are directed to `index.php` by your web server configuration. This file:

- Starts output buffering (`ob_start()`).
- Requires the framework initialization script (`System/Core/init.php`).
- Resolves the `Core\App` class from the Dependency Injection Container.
- Runs the application: `$app->run()`.
- Flushes the output buffer (`ob_end_flush()`).

## Initialization

The initialization script `System/Core/init.php` prepares the framework ecosystem:

- **PHP Version Check**: Ensures at least PHP 8.2 is running.
- **Environment Setup**: Configures `max_execution_time` for non-CLI requests and sets `umask(0000)`.
- **Autoloading**: Registers the custom framework `Autoloader` using `DirectoryDiscovery`.
- **Kernel Boot**: Initializes the `Core\Kernel` and calls `boot()`, which:
  - Runs the `Bootstrapper` to load **Environment** variables (`.env`), **Composer** dependencies (`vendor/autoload.php`), and **Global Helpers**.
  - Registers the **ErrorHandler**.
  - Registers and boots **Service Providers**.
  - Sets the application **Timezone**.

## The Application Run

The `run()` method in (`Core\App`) orchestrates the main flow:

- **Request Capture**: The `Request` object is resolved from the container and synchronized with the router.
- **Routing**: The `UrlResolver` matches the URL to a controller and method.
- **Security Check**: Validates security tokens (CSRF) for state-changing requests.
- **Middleware Pipeline**: Passes the request through the middleware stack matched to the route.
- **Dispatching**: The controller action is executed via the container's `call()` method.
- **Response**: The controller returns a `Response` object which is then sent to the client.

## Termination

After the response is sent (via `Response::complete()`), the application enters the **Termination** phase. This is vital for maintaining stability in long-running processes like queue workers.

- **Events**: The application dispatches termination events:
  - `Core\Events\KernelTerminateEvent` for web requests.
  - `Core\Events\ConsoleTerminateEvent` for CLI commands.
- **Listeners**: Registered listeners execute, most notably the `ProcessDeferredTasksListener` which handles tasks pushed via the `defer()` helper.
- **State Reset**: The Kernel triggers `terminate()` which:
  - Calls `terminate()` on all loaded **Service Providers** implementing `TerminableInterface`.
  - **Clears Static State**: Resets query logs (`Connection`), benchmarks (`Benchmark`), route caches (`UrlResolver`), and internal inflector caches (`Inflector`).

```php
// Deferred task example
defer(function() {
    // This runs after the response is sent
});
```
