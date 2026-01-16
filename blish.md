# Blish

**Blish** is a lightweight newsletter and email marketing package for the Anchor Framework. It allows you to manage subscribers, create campaigns, and track engagement.

## Core Capabilities

- **Subscriber Management**: Add, update, and unsubscribe users.
- **Campaigns**: Create and send email campaigns to your subscribers.
- **Analytics**: Track email opens and link clicks (if configured).

## Installation

```bash
php dock package:install Blish --packages
```

## Basic Usage

### Managing Subscribers

Blish provides a fluent interface for managing subscribers.

```php
use Blish\Blish;

// Subscribe a new user
$subscriber = Blish::subscribe('user@example.com', [
    'name' => 'John Doe',
    'source' => 'web_footer',
]);

// Find a subscriber
$user = Blish::find('user@example.com');

// Unsubscribe a user
Blish::unsubscribe('user@example.com');
```

### Advanced Subscriber Builder

For more control, you can use the fluent subscriber builder:

```php
use Blish\Blish;

Blish::subscriber()
    ->email('jane@example.com')
    ->name('Jane Doe')
    ->data(['interests' => ['tech', 'news']])
    ->active() // Sets status to 'active'
    ->save();
```

### Creating Campaigns

You can create campaigns using the campaign builder:

```php
use Blish\Blish;
use Helpers\DateTimeHelper;

// 1. Create and schedule the campaign
$campaign = Blish::campaign()
    ->title('Monthly Newsletter')
    ->subject('Hello {name}!')
    ->meta(['content' => '<h1>Welcome</h1>'])
    ->schedule(DateTimeHelper::now()->addHour()) // Fluent scheduling
    ->create();

// Alternatively, schedule later:
// $campaign->update(['status' => 'scheduled', ...]);
```

> **Note**: For scheduled campaigns to be sent, you must schedule the `blish:process` command to run frequently (e.g., every minute) in your server's crontab or via the framework's scheduler:
>
> ```bash
> * * * * * php /path/to/project/dock blish:process
> ```

## Analytics

Blish provides a service to track engagement (opens, clicks, sends).

### Tracking Events

You can record events programmatically:

```php
use Blish\Blish;

// Record that an email was opened
Blish::analytics()->recordOpen($subscriber, $campaign);

// Record a link click
Blish::analytics()->recordClick($subscriber, $campaign, 'https://example.com');
```

Note that `recordSent` is automatically called by the `CampaignProcessorService`.

### Retrieving Stats

Access analytics stats via the database models:

````php
// Get stats for a campaign
$stats = $campaign->stats;
// $stats->opens, $stats->clicks, $stats->sent

### Trend Analysis

You can retrieve engagement trends (e.g., daily, hourly) directly from the `Campaign` model:

```php
// Get daily open trends
$openTrends = $campaign->getOpenTrend('day');
// Returns ['2026-01-01' => 10, '2026-01-02' => 15, ...]

// Get hourly click trends
$clickTrends = $campaign->getClickTrend('hour');
// Returns ['2026-01-01 08:00:00' => 2, '2026-01-01 09:00:00' => 5, ...]

// Supported intervals: 'day', 'hour', 'month'
````

## Configuration

Configuration is located in `App/Config/blish.php`.

```php
return [
    'subscriber' => [
        'verify' => true, // Use double opt-in by default
    ],

    'campaign' => [
        'throttle' => 100, // Emails per minute
    ],
];
```

## Integrations

Blish integrates with other core packages:

- **Link**: Used to generate signed unsubscribe URLs implementation.
- **Mail**: Used to dispatch campaign emails via configured mail drivers.
