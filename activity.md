# Activity

The Activity package provides a high-performance, behavioral tracking and product analytics system for the Anchor Framework. It is engineered to offer deep insights into user journeys, conversion funnels, and system audits.

## Features

- **Fluent Logging**: Expressive, chainable API for capturing rich activity data.
- **Polymorphic Tracking**: Associate activities with any system resource (e.g., Order, Article, User).
- **Behavioral Analytics**: Built-in engine for tracking DAU/MAU, stickiness, and conversion funnels.
- **Context Awareness**: Automatic capture of IP, User-Agent, Session, and Channel metadata.
- **Structured Metadata**: Secure JSON storage for custom payloads and interpolated descriptions.
- **Retention Management**: Automated pruning utilities to maintain database performance.
- **Presentation Ready**: Dedicated ViewModels for consistent UI integration.

## Installation

Activity is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Activity --packages
```

This will automatically:

- Run the migration for Activity tables.
- Register the `ActivityServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/activity.php`

```php
return [
    // Default logging level
    'default_level' => 'info',

    // Retention policy (days)
    'retention_days' => 30,

    // Automatic context capture settings
    'capture' => [
        'ip' => true,
        'user_agent' => true,
        'session' => true,
        'channel' => true,
    ],
];
```

## Basic Usage

### Static Facade

The `Activity\Activity` facade is the primary interface for logging.

```php
use Activity\Activity;

// Simple log
Activity::log($userId, 'User completed the onboarding tour');

// Fluent logging with context
Activity::description('Processed payment for {order_ref}')
    ->user($userId)
    ->subject($order)
    ->data(['order_ref' => 'ORD-987'])
    ->tag('billing')
    ->level('critical')
    ->log();
```

### Global Helper

For rapid development, use the `activity()` helper:

```php
activity('Updated profile', user_id: $user->id);
```

## Advanced Features

### Behavioral Intelligence

Access the analytics engine via `Activity::analytics()`. It uses optimized SQL aggregations to handle large datasets.

```php
// 1. Retention Metrics
$retention = Activity::analytics()->getRetentionStats();
// Returns: { dau: 120, mau: 3000, stickiness: '4%' }

// 2. Conversion Funnels
$funnel = Activity::analytics()->getFunnel(['signup', 'verify', 'first_post']);
// Returns drop-off analysis between each step.

// 3. Traffic Distribution
$channels = Activity::analytics()->getChannelStats();
// Returns breakdown by Web, API, and CLI.
```

## Use Case Walkthrough

### Scenario: Security Monitoring Workflow

#### Capture Failed Login Attempt

```php
// AuthController.php
public function login()
{
    if (!$this->auth->attempt($credentials)) {
        Activity::description('Failed login attempt for {email}')
            ->data(['email' => $credentials['email']])
            ->tag('security')
            ->level('warning')
            ->metadata(['reason' => 'invalid_password'])
            ->log();
    }
}
```

#### Monitor Security Trends

```php
// AdminDashboardController.php
public function securityMetrics()
{
    return Activity::analytics()
        ->where('tag', 'security')
        ->getActionBreakdown('daily', 7);
}
```

## Package Integrations

### Audit Package

While Activity tracks behavioral trends, it integrates with `Audit` for tamper-proof compliance logs.

```php
// Log behavioral activity
Activity::log($user->id, 'Exported payroll data');

// Log compliance audit
Audit::make()
    ->event('payroll.export')
    ->on($user)
    ->log();
```

## Troubleshooting

| Error/Log                 | Cause                                     | Solution                                           |
| :------------------------ | :---------------------------------------- | :------------------------------------------------- |
| `TypeError: user_id`      | Logging without a session or explicit ID  | Ensure `user_id` is provided or user is logged in. |
| "No analytics data"       | Empty `activity` table                    | Populate the table with test log entries.          |
| Performance lag on stats  | Table has grown too large                 | Run the pruning command to clear old logs.         |

## Service API Reference

### Activity (Facade)

| Method                                       | Description                                            |
| :------------------------------------------- | :----------------------------------------------------- |
| `description(string $text)`                  | Sets the activity description (supports {tokens}).     |
| `data(array $data)`                          | Sets data for description interpolation.               |
| `user(int $userId)`                          | Associates an actor with the activity.                 |
| `subject(BaseModel $obj)`                    | Sets the polymorphic subject (resource).               |
| `tag(string $tag)`                           | Categorizes the activity for filtering.                |
| `level(string $level)`                       | Sets severity (info, warning, critical).               |
| `metadata(array $meta)`                      | Attaches raw JSON context.                             |
| `log()`                                      | Persists the activity record.                          |
| `listUserActivities(User $user, int $page)`  | Paginated list of activities for a specific user.      |
| `listRecentActivities(int $days, int $page)` | Paginated list of recent activities across the system.  |
| `getSummary(Activity $activity)`             | Generates a human-readable summary with relative time. |
| `analytics()`                                | Returns the `ActivityAnalytics` service.               |

### ActivityAnalytics

| Method                       | Description                                      |
| :--------------------------- | :----------------------------------------------- |
| `getTrends($period, $limit)` | Returns overall volume over time.                |
| `getRetentionStats()`        | Calculates DAU/MAU and Stickiness.               |
| `getFunnel(array $steps)`    | Performs conversion and drop-off analysis.      |
| `getChannelStats()`          | Returns traffic distribution (Web, API, CLI).    |
| `getUserBehavior($userId)`   | Returns a behavior profile for a specific user.  |
| `getTagStats()`              | Returns distribution by activity tags.           |

## Console Commands

| Command                 | Description                                    |
| :---------------------- | :--------------------------------------------- |
| `activity:prune`        | Deletes logs older than the retention period.  |
| `activity:stats`        | Provides a quick CLI snapshot of log volume.   |

## Automation

The Activity package automatically registers its pruning command in the scheduler. This is defined in:

```php
// packages/Activity/Schedules/ActivityPruneSchedule.php
namespace Activity\Schedules;
 
use Cron\Interfaces\Schedulable;
use Cron\Schedule;
 
class ActivityPruneSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('activity:prune')
            ->daily();
    }
}
```

### Automated Audit Logging

Rank uses the **Route Context** system to automatically log security-sensitive and state-changing actions without any manual code in your controllers.

Every successful `POST`, `PUT`, or `DELETE` request is automatically audited with:
- **Human Description**: Auto-generated based on entity and action (e.g., "UPDATE action on Profile").
- **Actor Tracking**: Automatically associates with the authenticated user.
- **Data Privacy**: Masked metadata (passwords and tokens are automatically scrubbed).
- **Environmental Context**: Automatically captures IP, URL, and User-Agent.

For more details, see the [Route Context](route-context.md) documentation.

## Security Best Practices

- **Mask Sensitive Data**: Never include passwords, credit card numbers, or PII in the `data` or `metadata` payloads.
- **Context Verification**: Always verify that the `user_id` associated with a log matches the authenticated identity to prevent audit spoofing.
- **Automated Pruning**: The system automatically handles `activity:prune` via the central scheduler to maintain optimal database performance.
