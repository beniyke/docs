# Request Lifecycle

Understanding the request lifecycle is crucial for mastering Anchor. It describes the path a request takes from entering your application to the final response.

## Entry Point

All requests are directed to `index.php` by your web server configuration. This file:

- Requires the framework initialization script (`System/Core/init.php`).
- The initialization script registers the framework autoloader, which then loads Composer dependencies if available.
- Resolves the `Core\App` class from the container.
- Runs the application: `$app->run()`.

## Initialization

The initialization script `System/Core/init.php` sets up the environment:

- **Autoloading**: Registers the custom framework autoloader.
- **Container**: Initializes the Dependency Injection Container.
- **Kernel**: Boots the core kernel, which:
  - Loads **Environment** variables (`.env`).
  - Loads **Composer** dependencies and global helpers.
  - Registers **Error Handling** and exception handlers.
  - Loads **Service Providers** and sets the **Timezone**.

## The Application Run

The `run()` method in (`Core\App`) orchestrates the main flow:

- **Request Capture**: The `Request` object is created and injected into the `App` instance during container resolution.
- **Routing**: The `Router` matches the URL to a controller and method.
- **Security Checks**: Validates CSRF tokens and state-changing requests.
- **Middleware**: Passes the request through the middleware stack matched to the route (e.g., Web or API middleware).
- **Dispatching**: The controller action is executed.
- **Response**: The controller returns a `Response` object.
- **Sending**: The response headers and content are sent to the browser.

## Termination

After the response is sent, the application executes any **deferred tasks**. This allows for heavy operations (like sending emails or logging) to happen without delaying the response to the user.

```php
defer(function() {
    // This runs after the response is sent
});
```
