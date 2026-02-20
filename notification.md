# Notification

The Notification package provides an infrastructure for database-backed in-app notifications. It features a dedicated delivery channel for the core [Notify](notify.md) system, comprehensive behavioral analytics, and automated maintenance utilities.

## Features

- **Fluent Notifications**: Easy-to-use API for delivering targeted messages to users.
- **Global Delivery**: Capability to broadcast announcements to the entire user base.
- **Behavioral Analytics**: Integrated engine for tracking notification trends, read rates, and label distribution.
- **Smart Pruning**: Automated cleanup of old notification data with customizable retention policies.
- **Cache-Native**: Deep integration with Anchor's cache tagging for instant UI updates.
- **Secure Management**: Granular control over notification states (read, viewed, cleared).

## Installation

Notification is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Notification --packages
```

This will automatically:

- Run the migration for Notification tables.
- Register the `NotificationServiceProvider`.
- Enable the `in-app` channel within the core [Notify](notify.md) system.

## Basic Usage

### Notification Facade

The `Notification\Notification` facade is the primary interface for managing notifications in your application.

```php
use Notification\Notification;

// 1. Sending Notifications
Notification::notifyUser([
    'user_id' => $userId,
    'message' => 'Your order has been shipped!',
    'label' => 'shipping',
    'url' => '/orders/123'
]);

// 2. Global Broadcasts
Notification::notifyAll([
    'message' => 'System maintenance scheduled for tonight.',
    'label' => 'announcement'
]);

// 3. User Inbox Management
Notification::listNotifications($user); // Returns Paginator & marks as viewed
Notification::markAllAsRead($user);     // Mark all as read
Notification::clearUserNotifications($user); // Inbox Zero
```

## Sending via Notify

Once installed, the `in-app` channel is natively supported by the core [Notify](notify.md) system.

```php
use Notify\Notify;

Notify::inApp(OrderShippedAlert::class, $payload);
```

## Model Integration

### The Notification Model

The `Notification\Models\Notification` model provides static helpers and query scopes for advanced management.

```php
use Notification\Models\Notification;

// Static Helpers
$unread = Notification::unreadForUser($userId);
$read = Notification::readForUser($userId);
$recent = Notification::recentForUser($userId);
$count = Notification::unreadCountForUser($userId);

// Query Scopes
$highPriority = Notification::unread()->where('label', 'urgent')->get();

// Instance Methods
$notification->markAsRead();
```

## Use Case Walkthrough

#### Scenario: Real-time User Engagement

**Delivering a Reward Notification**

```php
// RewardService.php
public function rewardUser($user, $points)
{
    Notification::notifyUser([
        'user_id' => $user->id,
        'message' => "You've earned {$points} loyalty points!",
        'label' => 'reward',
        'url' => '/profile/rewards'
    ]);
}
```

**Displaying the Notification Bell**

```php
// HeaderViewModel.php
public function getNotificationStats()
{
    return [
        'count' => Notification::unreadCountForUser($this->user->id),
        'items' => Notification::recentForUser($this->user->id, 5)
    ];
}
```

## Analytics & Reporting

The `NotificationAnalytics` service provides deep insights into delivery and engagement patterns.

```php
// Volume Trends
$trends = Notification::analytics()->getTrends('daily', 30);
// Returns: [['date' => '2026-01-30', 'count' => 15], ...]

// Distribution by Label
$breakdown = Notification::analytics()->getLabelBreakdown();
// Returns: [['label' => 'alert', 'count' => 50], ...]

// Read Rate Metrics
$stats = Notification::analytics()->getReadRateStats();
// Returns: ['total' => 100, 'read' => 80, 'unread' => 20, 'read_rate' => '80%']
```

## Troubleshooting

| Error/Log                      | Cause                                  | Solution                                     |
| :----------------------------- | :------------------------------------- | :------------------------------------------- |
| `Table 'notification' not found`| Migration has not been run             | Run `php dock package:install Notification`  |
| Notifications not clearing     | Cache invalidation failure             | Verify `notifications` cache tag is enabled  |
| Analytics returning empty      | No notification data exists            | Send test notifications to populate stats    |

## Service API Reference

### Notification (Facade)

| Method                          | Description                                      |
| :------------------------------ | :----------------------------------------------- |
| `notifyUser(array $data)`       | Delivers a notification to a specific user.      |
| `notifyAll(array $data)`        | Broadcasts a notification to all users.          |
| `listNotifications($user, ...)` | Returns paginated notifications for a user.      |
| `markAllAsRead($user)`          | Marks all user notifications as read.            |
| `clearUserNotifications($user)` | Deletes all notifications for a user.            |
| `analytics()`                   | Returns the `NotificationAnalytics` service.     |

### Notification (Model)

| Scope / Helper                  | Description                                      |
| :------------------------------ | :----------------------------------------------- |
| `unread()`                      | Scope to filter unread notifications.            |
| `read()`                        | Scope to filter read notifications.              |
| `unreadForUser($userId)`        | Static helper to fetch unread notifications.     |
| `readForUser($userId)`          | Static helper to fetch read notifications.       |
| `unreadCountForUser($userId)`   | Returns total unread count for a user.           |

## Console Commands

| `notification:prune`    | Automatically deletes old notification data    |
 
## Automation
 
The Notification package includes an automated pruning task that is registered in the framework scheduler. This ensures old notification data is cleaned up based on your retention policy.
 
```php
// packages/Notification/Schedules/NotificationPruneSchedule.php
namespace Notification\Schedules;
 
use Cron\Interfaces\Schedulable;
use Cron\Schedule;
 
class NotificationPruneSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('notification:prune')
            ->daily();
    }
}
```

## Security Best Practices

- **User Isolation**: Always ensure `user_id` is derived from authenticated sessions when performing state updates.
- **Automated Retention**: The system automatically handles `notification:prune` via the central scheduler to prevent database bloat.
- **Labeling**: Use consistent labels (e.g., `security`, `billing`) to allow users to filter notification types in your UI.
