# Permit

The Permit package provides an Authorization system , for Anchor applications featuring hierarchical RBAC (Role-Based Access Control) and granular ABAC (Attribute-Based Access Control).

> Permit is designed to be "Secure by Default". If no permission or gate is found, access is automatically denied.

## Features

- **Hierarchical Roles**: Roles can inherit permissions from parent roles (e.g., Admin inherits from Editor).
- **Granular Permissions**: Define specific abilities and group them for easy management.
- **Dynamic Gates**: Custom closure-based authorization for complex business logic (ABAC).
- **Direct User Overrides**: Grant or deny permissions directly to users outside of their roles.
- **Super Admin Bypass**: Built-in support for a master role that automatically bypasses all permission checks.
- **High Performance**: Optimized permission caching ensures zero-latency checks in production.
- **Smart Middleware**: Decouple logic by automatically deriving permissions from URI segments.

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
- Register `CheckPermissionMiddleware` for `web` and `api` groups.

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

Avoid permission duplication by nesting roles. This creates a chain of command where high-level roles automatically possess the capabilities of lower-level ones.

> [!NOTE]
> **Inheritance is Additive**: By default, a child role receives *all* permissions from its parent. There is no "subtractive inheritance" at the role level.

**Real-World Scenario: Media CMS**
- `Guest`: Can `view.posts`
- `Editor`: Inherits `Guest` + `create.posts`, `edit.posts`
- `Manager`: Inherits `Editor` + `publish.posts`, `delete.posts`
- `Admin`: Inherits `Manager` + `manage.users`, `manage.settings`

```php
Permit::role()
    ->slug('admin')
    ->inherits('editor')
    ->permissions(['settings.manage'])
    ->create();
```

### Removing or "Replacing" Permissions

If you need a role that is "like an Editor but without delete powers", you have two options:

- **Direct User Override (Recommended)**: If it's for a specific person, use `$user->denyPermissionTo('posts.delete')`.
- **Flat Role Definition**: Create a new role that doesn't inherit, and manually assign the subset of permissions.
- **Mid-level Inheritance**: Create a `BasicEditor` role with common powers, then have `FullEditor` (with delete) and `RestrictedEditor` (without delete) inherit from `BasicEditor`.

### Custom Gates

Define complex logic that goes beyond simple permissions (ABAC). Gates are ideal for implementing "Ownership" logic or time-based restrictions.

> Custom gates should be defined in the `boot()` method of a Service Provider, for example `App\Providers\AuthServiceProvider.php`.

**Real-World Scenario: Resource Ownership**
Only the creator of a post (or an admin) should be able to edit it.

```php
Permit::define('update-post', function ($user, $post) {
    return $user->id === $post->author_id || $user->hasRole('admin');
});

// Using the gate in a Controller
if (Permit::can($user, 'update-post', $post)) {
    // Current user is the author or an admin
}
```

### Direct User Negation (Overrides)

While roles handle bulk permissions, sometimes you need to override a role for a specific individual. Permit allows you to **Grant** or **Deny** permissions directly to a user.

> Direct **Deny** takes absolute precedence. Even if a user's role grants them a permission, if it is explicitly denied on the user level, `can()` will return `FALSE`.

**Real-World Scenario: Suspended Privileges**
An "Editor" is currently under review and should not be allowed to delete posts, even though the 'editor' role usually allows it.

```php
$user = User::find(123);

// Explicitly take away the ability
$user->denyPermissionTo('posts.delete');

// This now returns FALSE, ignoring role inheritance
if ($user->can('posts.delete')) { ... }
```

### Permission Analytics

Monitor access patterns and security health. This is vital for auditing which permissions are being used and identifying potential security risks.

```php
$analytics = Permit::analytics();

// Real-world: Get distribution of roles across the user base
$distribution = $analytics->getRoleDistribution();

// Real-world: Identify most common authorization failures (potential attacks)
$failures = $analytics->getFailureLogs(10);

// Real-world: Get permissions grouped by module
$groups = $analytics->getPermissionsByGroup();
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

| Method                    | Description                                     |
| :------------------------ | :---------------------------------------------- |
| `role()`                  | Starts a fluent `RoleBuilder`.                  |
| `permission()`            | Starts a fluent `PermissionBuilder`.            |
| `can($user, $ability)`    | Evaluates if a user has authorization.          |
| `cannot($user, $ability)` | Inverse of `can()`.                             |
| `hasAnyRole($user, $arr)` | Checks if user has any of the listed roles.     |
| `hasAllRoles($user, $arr)`| Checks if user has all of the listed roles.     |
| `define($name, $closure)` | Registers a custom authorization gate.          |
| `analytics()`             | Returns the `PermitAnalytics` service.          |
| `clearCache()`            | Wipes the permission cache.                     |

### Permission Builder (Fluent)

Access via `Permit::permission()`.

| Method          | Description                                    |
| :-------------- | :--------------------------------------------- |
| `slug($slug)`   | Sets the unique identifier for the permission. |
| `name($name)`   | Sets a human-readable name.                    |
| `group($group)` | Organizes permissions into UI categories.      |
| `create()`      | Persists the permission to the database.       |

### HasRoles / HasPermissions (Traits)

| Method                       | Description                                |
| :--------------------------- | :----------------------------------------- |
| `assignRole($slug)`          | Grants a role to the user.                 |
| `removeRole($slug)`          | Removes a role from the user.              |
| `hasRole($slug)`             | Checks if the user has a specific role.    |
| `can($ability)`              | Standard Laravel-style permission check.   |
| `givePermissionTo($slug)`    | Grants a direct (un-rolled) permission.    |
| `denyPermissionTo($slug)`    | Explicitly denies a permission (override). |
| `revokePermissionTo($slug)`  | Removes a direct grant or deny.            |
| `getAllPermissions()`        | Returns an array of all Permission models. |
| `getPermissionNames()`       | Returns an array of permission slugs.      |

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

## Smart Middleware

Permit includes a **Smart Middleware** that automatically enforces permissions based on URI conventions.

### How it works

The middleware scans URI segments from **right to left** to find a recognized **action** (mapped in config). Once found, it assumes the preceding segment is the **resource**.

**Example:**

- `account/user/create` → Action: `create`, Resource: `user` → Permission: `users.create` (automatically pluralized).

### Action Mapping

Customize behavior in `App/Config/permit.php`:

```php
'smart_middleware' => [
    'enabled' => true,
    'action_map' => [
        'create'  => 'create',
        'store'   => 'create',
        'edit'    => 'edit',
        'update'  => 'edit',
        'destroy' => 'delete',
        'index'   => 'manage',
    ],
],
```

### Route Context Integration

When "Smart Middleware" is enabled, it automatically hydrates the `Request` object with route context. This data can be retrieved anywhere in your application (Controllers, Views, etc.).

```php
// Get the structured permission (e.g., 'users.create')
$permission = $request->getRoutePermission();

// Get specific resource/action
$resource = $request->getRouteContext('resource'); // 'users'
$action = $request->getRouteContext('action');     // 'create'
```

### View Engine Helpers

The metadata is also exposed directly in your view templates via private methods in the `ViewEngine`.

- `$this->canAccessAction(string|array $action)`: Checks if the user has permission for the specified action(s) on the *current* resource.
- `$this->isResourceActive(string $name)`: Returns 'active' if the current resource matches the name.
- `$this->getRouteTitle()`: Retrieves a human-readable title from `permit.titles` config.
- `$this->getBreadcrumbs()`: Generates a standard breadcrumb array based on context.

**Example:**

```php
<li class="<?= $this->isResourceActive('users') ?>">Users</li>

<?php if ($this->canAccessAction(['edit', 'delete'])): ?>
    <button>Manage Record</button>
<?php endif; ?>
```

This metadata enabling advanced features such as context-aware logging, automatic sidebar highlighting, and dynamic breadcrumbs without manual URL parsing.

## Automation

The Permit package includes an automated cache warming task that is registered in the framework scheduler. This ensures permission data is efficiently cached for performance.

```php
// packages/Permit/Schedules/PermitCacheSchedule.php
namespace Permit\Schedules;

use Cron\Interfaces\Schedulable;
use Cron\Schedule;

class PermitCacheSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('permit:cache')
            ->daily();
    }
}
```

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
