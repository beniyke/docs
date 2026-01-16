# Rollout

The Rollout package provides a production-ready Feature Flag and A/B Testing system for the Anchor Framework. It enables progressive feature releases, user-segment targeting, and controlled experimentation.

## Features

- **Progressive Rollouts**: Gradually release features to a percentage of users (e.g., 5%, 25%, 100%).
- **Smart Targeting**: Enable features based on user roles (via `Permit` integration), email domains, or specific IDs.
- **Dynamic Scheduling**: Set start and end dates for time-sensitive features or promotions.
- **User Segments**: Create reusable audience groups for consistent targeting across features.
- **Consistent Hashing**: Ensures a user has a stable experience (always seeing the same flag state).
- **A/B Testing**: Run experiments by serving different feature states to randomized groups.
- **Permit Integration**: Native support for checking feature flags against user roles and permissions.

## Installation

Rollout is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Rollout --packages
```

This will automatically:

- Run database migrations for features and targeting tables.
- Register the `RolloutServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/rollout.php`

```php
return [
    'enabled' => env('ROLLOUT_ENABLED', true),
    'cache' => [
        'enabled' => true,
        'ttl' => 300,
    ],
    'default_state' => false,
];
```

## Basic Usage

### Create a Feature

```php
use Rollout\Rollout;

// Define a new feature
Rollout::feature()
    ->slug('new-checkout-flow')
    ->name('Modern Checkout')
    ->percentage(10) // 10% of users
    ->forRoles(['beta-testers'])
    ->create();
```

### Check Feature State

```php
if (Rollout::isEnabled('new-checkout-flow')) {
    // Show the new flow
}

// Or check for a specific user
if (Rollout::isEnabled('new-checkout-flow', $user)) {
    // ...
}
```

## Use Cases

#### Tiered Beta Rollout

Release a high-impact feature first to your internal team, then to Platinum resellers, and finally to 10% of the general public.

#### Implementation (In a Seeder or Console)

```php
use Rollout\Rollout;

// Step 1: Internal Staff & Beta Segments
Rollout::feature()
    ->slug('v2-dashboard')
    ->name('NexGen Dashboard')
    ->description('The new reactive dashboard interface')
    ->forDomains(['anchor-framework.com']) // Internal staff
    ->forRoles(['platinum-partner', 'beta-tester']) // Roles managed by Permit
    ->percentage(10) // Plus 10% of random traffic
    ->create();
```

#### Usage in View/Controller

```php
if (Rollout::isEnabled('v2-dashboard')) {
    return view('dashboard.v2');
}
```

## Package Integrations

### Permit (Authorization)

Rollout automatically detects if the `Permit` package is installed. When you use `forRoles()`, it uses `Permit`'s `hasRole()` logic under the hood to evaluate targeting.

```php
// This check works seamlessly with Permit roles
Rollout::feature()->forRoles(['admin', 'editor'])->create();
```

### Audit (Activity Logs)

Every time a feature flag is enabled, disabled, or its percentage is changed via the `Rollout` facade, an entry is automatically created in the `Audit` log.

## Service API Reference

### Rollout (Facade)

| Method                    | Description                                 |
| :------------------------ | :------------------------------------------ |
| `feature()`               | Starts a fluent `FeatureBuilder`.           |
| `isEnabled($slug, $user)` | Checks if a feature is active for a user.   |
| `enable($slug)`           | Globally enables a feature (100%).          |
| `disable($slug)`          | Globally disables a feature (0%).           |
| `setPercentage($s, $p)`   | Updates the rollout percentage dynamically. |
| `analytics()`             | Returns the `RolloutAnalytics` service.     |

### HasFeatures (Trait)

| Method                  | Description                                  |
| :---------------------- | :------------------------------------------- |
| `hasFeature($slug)`     | Direct check on the model instance.          |
| `hasAnyFeature($slugs)` | True if any of the features are enabled.     |
| `getEnabledFeatures()`  | Returns a list of all features the user has. |

### Feature (Model)

| Attribute    | Type       | Description                              |
| :----------- | :--------- | :--------------------------------------- |
| `is_active`  | `boolean`  | Global kill-switch status.               |
| `percentage` | `integer`  | Current rollout percentage (0-100).      |
| `starts_at`  | `datetime` | When the feature should activate.        |
| `rules`      | `json`     | targeting rules (roles, domains, users). |

## Troubleshooting

| Error/Log               | Cause                            | Solution                                   |
| :---------------------- | :------------------------------- | :----------------------------------------- |
| Flag state inconsistent | Cache TTL is high.               | Clear cache or reduce `rollout.cache.ttl`. |
| Feature not appearing   | `starts_at` is in the future.    | Check server time and feature schedule.    |
| "0% Rollout"            | Feature created but not enabled. | Set percentage or call `enable()`.         |

## Security Best Practices

- **Sensitive Flags**: Never expose a list of all available feature flags to the frontend; only return the states for the current user.
- **Dead-Code Removal**: Periodically audit and remove code paths for fully rolled-out or abandoned features.
- **Default Deny**: Always assume a feature is `false` if the database is unreachable or the slug is missing.
