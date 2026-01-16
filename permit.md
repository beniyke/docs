# Permit

The Permit package provides an Authorization system , for Anchor applications featuring hierarchical RBAC (Role-Based Access Control) and granular ABAC (Attribute-Based Access Control).

## Features

- **Hierarchical Roles**: Roles can inherit permissions from parent roles (e.g., Admin inherits from Editor).
- **Granular Permissions**: Define specific abilities and group them for easy management.
- **Dynamic Gates**: Custom closure-based authorization for complex business logic.
- **Direct User Overrides**: Grant or deny permissions directly to users outside of their roles.
- **Super Admin Bypass**: Built-in support for a master role (defined in config) that automatically bypasses all permission checks and gates, returning `TRUE` for every request.
- **High Performance**: Optimized permission caching to ensure zero-latency checks in production.

## Installation

Permit is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Permit --packages
```

This will automatically:

- Run database migrations for roles, permissions, and assignments.
- Register the `PermitServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/permit.php`

```php
return [
    'super_admin_role' => 'super-admin',
    'role_hierarchy' => true,
    'cache' => [
        'enabled' => true,
        'ttl' => 3600,
    ],
    'user_model' => App\Models\User::class,
];
```

## Basic Usage

### Setup Models

Add the necessary traits to your User model:

```php
use Permit\Traits\HasRoles;
use Permit\Traits\HasPermissions;

class User extends BaseModel
{
    use HasRoles, HasPermissions;
}
```

### Configure Access

```php
use Permit\Permit;

// Create role with permissions
Permit::role()
    ->slug('editor')
    ->name('Content Editor')
    ->permissions(['posts.create', 'posts.edit'])
    ->create();

// Assign to user
$user->assignRole('editor');
```

### Check Authorization

```php
// Via model trait
if ($user->can('posts.create')) {
    // Process creation
}

// Via Facade
if (Permit::can($user, 'users.manage')) {
    // ...
}
```

## Advanced Features

### Role Inheritance

Avoid permission duplication by nesting roles:

```php
Permit::role()
    ->slug('admin')
    ->inherits('editor')
    ->permissions(['settings.manage'])
    ->create();
```

### Custom Gates

Define complex logic that goes beyond simple permissions (ABAC).

> Custom gates should be defined in the `boot()` method of a Service Provider, for example `App\Providers\AuthServiceProvider.php` or other custom provider

```php
Permit::define('update-post', function ($user, $post) {
    return $user->id === $post->author_id || $user->hasRole('admin');
});

// Using the gate in a Controller
if (Permit::can($user, 'update-post', $post)) {
    // Current user is the author or an admin
}
```

### Permission Analytics

Monitor access patterns and security health:

```php
$analytics = Permit::analytics();

// Get permission usage density
$stats = $analytics->getUsageMetrics();

// Identify most common authorization failures
$failures = $analytics->getFailureLogs(10);
```

## Implementation Flow

Here is how you would implement a full authorization cycle for a Blog application.

### Define Permissions

`App/storage/database/seeds/RolesSeeder.php`

> This file doesn't exist by default. It is a recommended location to manage your authorization "Source of Truth". You can create it manually to define your roles and permissions in code.

Use the builder to create grouped permissions for clarity.

```php
use Permit\Permit;

// Define permissions
Permit::permission()->slug('posts.create')->group('Writing')->create();
Permit::permission()->slug('posts.edit')->group('Writing')->create();
Permit::permission()->slug('posts.delete')->group('Admin')->create();
```

### Define Roles & Inheritance

Configure the hierarchical structure.

```php
// Editor can write and edit
Permit::role()
    ->slug('editor')
    ->permissions(['posts.create', 'posts.edit'])
    ->create();

// Admin inherits Editor powers and adds deletion
Permit::role()
    ->slug('admin')
    ->inherits('editor')
    ->permissions(['posts.delete'])
    ->create();
```

### Register Business Logic Gates (AuthServiceProvider.php)

Handle resource-specific ownership.

```php
Permit::define('manage-account', function ($user, $targetUser) {
    return $user->id === $targetUser->id;
});
```

### Assign and Verify

```php
// Assigning during registration
$user->assignRole('editor');

// Checking in a Middleware or Controller
if ($user->can('posts.delete')) {
     // This will return FALSE for Editor, TRUE for Admin
}
```

## Service API Reference

### Permit (Facade)

| Method                    | Description                            |
| :------------------------ | :------------------------------------- |
| `role()`                  | Starts a fluent `RoleBuilder`.         |
| `permission()`            | Starts a fluent `PermissionBuilder`.   |
| `can($user, $ability)`    | Evaluates if a user has authorization. |
| `define($name, $closure)` | Registers a custom authorization gate. |
| `analytics()`             | Returns the `PermitAnalytics` service. |

### Permission Builder (Fluent)

Access via `Permit::permission()`.

| Method          | Description                                    |
| :-------------- | :--------------------------------------------- |
| `slug($slug)`   | Sets the unique identifier for the permission. |
| `name($name)`   | Sets a human-readable name.                    |
| `group($group)` | Organizes permissions into UI categories.      |
| `create()`      | Persists the permission to the database.       |

### HasRoles / HasPermissions (Traits)

| Method                    | Description                              |
| :------------------------ | :--------------------------------------- |
| `assignRole($slug)`       | Grants a role to the user.               |
| `hasRole($slug)`          | Checks if the user has a specific role.  |
| `can($ability)`           | Standard Laravel-style permission check. |
| `givePermissionTo($slug)` | Grants a direct (un-rolled) permission.  |

### Custom Logic (GateManager)

Access via `Permit::gates()`. This service manages closure-based authorization (Gates) and model-based Policies.

| Method                            | Description                                                              |
| :-------------------------------- | :----------------------------------------------------------------------- |
| `define($name, $closure)`         | Register a custom gate.                                                  |
| `check($ability, $user, $params)` | Manually evaluate a gate.                                                |
| `resource($name, $policyClass)`   | Automatically map CRUD abilities (view, update, etc.) to a Policy class. |
| `before($closure)`                | Register a callback that runs before all authorization checks.           |
| `after($closure)`                 | Register a callback that runs after all authorization checks.            |

### Command Line Interface

| Command                 | Description                                           |
| :---------------------- | :---------------------------------------------------- |
| `php dock permit:sync`  | Syncs permissions from config/code to the database.   |
| `php dock permit:cache` | Clears the authorization cache for immediate updates. |

### Role (Model)

| Attribute     | Type       | Description                                |
| :------------ | :--------- | :----------------------------------------- |
| `slug`        | `string`   | Unique identifier (e.g., 'admin').         |
| `permissions` | `relation` | BelongsToMany relationship to permissions. |
| `parent_id`   | `integer`  | ID of the parent role for inheritance.     |

## Troubleshooting

| Error/Log                 | Cause                                  | Solution                                 |
| :------------------------ | :------------------------------------- | :--------------------------------------- |
| Permission not reflecting | Cache hasn't expired yet.              | Run `php dock permit:cache` to clear.    |
| "Unauthorized" exception  | User lacks role or direct permission.  | Check assignments via `$user->roles()`.  |
| Cyclic inheritance error  | Role is trying to inherit from itself. | Verify the `inherits()` slug in builder. |

## Security Best Practices

- **Principle of Least Privilege**: Start with zero permissions and grant only what is necessary.
- **Prefer Roles**: Use roles for bulk management and direct permissions only for exceptional overrides.
- **Explicit Gates**: Use Gates for resource-specific logic (e.g., "owner of object") instead of generic "edit" permissions.
- **Audit Failures**: Monitor `analytics()->getFailureLogs()` to detect unauthorized access attempts.
