# Route Context

Route Context is a powerful, automated metadata system in the Anchor Framework that allows the application to "understand" its purpose at any given point in the request lifecycle. It powers a suite of **Smart Features** that reduce boilerplate and ensure consistency across your application.

## How it Works

The framework automatically hydrants the Route Context as soon as a request is resolved to a controller. It extracts standard identifiers from the controller's namespace and method:

- **Namespace**: `App\{Domain}\Controllers\{Entity}Controller`
- **Method**: `{Action}`

For example, `App\Account\Controllers\UserController@edit` results in:

- `domain`: `Account`
- `entity`: `User`
- `resource`: `users` (auto-pluralized)
- `action`: `edit`
- `validator`: `App\Account\Validations\Form\UserFormRequestValidation` (explicit override)

## Key Features Powered by Context

The Route Context system currently powers five major "Smart Features" across the framework:

### Smart Validation

Automatically resolves and executes the correct `RequestValidation` class based on current context. No need to manually instantiate or call validators in your controllers.

- See [Validation Documentation](validation.md#smart-validation)

### Smart Cache Clearing

Automatically flushes cache tags for the current `entity` whenever a state-changing request (POST/PUT/DELETE) is successful.

- See [Query Builder Cache](query-builder.md#smart-cache-clearing)

### Automated Audit Logging

Generates human-readable audit trails (e.g., "EDIT action on User in Account") automatically for every state-changing action.

- See [Activity Documentation](activity.md#automated-audit-logging)

### Granular Maintenance Mode

Allows locking down specific `resources` (e.g., "maintenance for the 'payments' resource") without taking the entire application offline.

- See [Firewall Documentation](firewall.md#granular-maintenance)

### Automated SEO (Rank)

Generates SEO-friendly page titles (e.g., "Edit User") and social metadata automatically based on the active resource and action.

- See [Rank Documentation](rank.md#automated-route-titles)

## Accessing Context

You can access or modify the context at any time via the `Request` object:

```php
// Get specific context
$resource = $request->getRouteContext('resource');

// Manually set or override context
$request->setRouteContext('custom_tag', 'value');
```

## Template Usage

Context is also available in your views for dynamic UI adjustments:

```php
<?php if ($this->isResourceContext('users')): ?>
    <!-- Active menu state logic -->
<?php endif; ?>
```

### Fluent Context Helpers

Instead of manually checking strings, Anchor provides fluent helpers in view templates:

- `$this->isResourceContext('users')`: Checks if `resource` is `users`.
- `$this->isActionContext('edit')`: Checks if `action` is `edit`.
- `$this->isDomainContext('Account')`: Checks if `domain` is `Account`.
- `$this->isEntityContext('User')`: Checks if `entity` is `User`.
- `$this->context('key', 'default')`: Retrieves any context value.

## Proactive Overrides (Zero-Controller)

You can explicitly define the route context in `App/Config/route.php`. This is useful for "Zero-Controller" setups where you want to set the context without any PHP logic.

### Use Case: 

#### The "Identity" Override

Standard Anchor routes resolve following a `{domain}/{entity}/{action}` pattern. For example, `auth/login` would normally resolve to:

- **Domain**: Auth
- **Entity**: Login
- **Resource**: logins
- **Action**: index

However, for a clean UI, you might want it to be treated as part of an "Identity" resource. By using a proactive override, you can force this mapping:

```php
return [
    'context' => [
        'auth/login' => [
            'resource' => 'identity',
            'action' => 'login',
        ],
    ],
];
```

### How it Works

- **Routing Phase**: When the `UrlResolver` matches a URL, it checks if a corresponding key exists in the `route.context` configuration.
- **Injection**: If found, these values are attached to the `RouteMatch` object.
- **Application Phase**: During `App::handle()`, the `hydrateRouteContext()` method is called. It first performs standard discovery (based on controller name) and then **merges** the explicit overrides, ensuring the config values take precedence.
- **Availability**: By the time your middleware or controller executes, `$request->getRouteContext()` will return the overridden values.

## Context Hydration Logic

The dynamic hydration logic in `App.php` automatically identifies the `domain`, `entity`, `resource`, and `action` based on the controller's namespace and method name if no explicit override is provided.
