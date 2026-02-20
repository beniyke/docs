# Blish

**Blish** is a lightweight newsletter and email marketing package for the Anchor Framework. It allows you to manage subscribers, create campaigns, and track engagement.

## Core Capabilities

- **Subscriber Management**: Add, update, and unsubscribe users.
- **Campaigns**: Create and send email campaigns to your subscribers.
- **Analytics**: Track email opens and link clicks (if configured).

## Installation

Blish is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Blish --packages
```

This command will:

- Publish the `blish.php` configuration file.
- Run the migration for Blish tables.
- Register the `BlishServiceProvider`.

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

### Retrieving Subscribers

You can retrieve subscribers using standard Eloquent methods on the `Subscriber` model.

```php
use Blish\Models\Subscriber;

// Get all subscribers
$allSubscribers = Subscriber::all();

// Filter by status (using scope)
$activeSubscribers = Subscriber::active()->get();

// Search by email/name (using scope)
$user = Subscriber::search('@company.com')->first();
```

### Segmentation with Lists and Tags

Blish supports organizing subscribers into lists and tagging them for targeted campaigns.

```php
// Add subscriber to a list
$subscriber->lists()->attach($listId);

// Add tags to a subscriber
$subscriber->tags()->attach($tagId);

// Get subscribers in a specific list (using scope)
$listSubscribers = Subscriber::inList($listId)->get();
```

### Managing Lists and Tags

You can manage lists and tags using standard Eloquent models.

```php
use Blish\Models\BlishList;
use Blish\Models\Tag;

// Create a new list
$list = BlishList::create([
    'name' => 'Weekly Newsletter',
    'slug' => 'weekly-newsletter',
    'description' => 'Updates every Monday',
]);

// Create a new tag
$tag = Tag::create([
    'name' => 'VIP',
    'slug' => 'vip',
]);
```

### Using Templates

Templates allow you to reuse email content across campaigns.

```php
use Blish\Models\Template;

// Create a template
$template = Template::create([
    'name' => 'Standard Layout',
    'content' => '<html>...</html>',
]);

// Use template in a campaign
$campaign = Blish::campaign()
    ->title('Campaign with Template')
    ->template($template->id)
    ->create();
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
    // ->inactive() // Or set status to 'inactive'
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

// $campaign->update(['status' => 'scheduled', ...]);
 ```
 
## Automation
 
The Blish package automatically handles campaign processing via the framework's central scheduler. This eliminates the need for manual cron management.
 
```php
// packages/Blish/Schedules/BlishSchedule.php
namespace Blish\Schedules;
 
use Cron\Interfaces\Schedulable;
use Cron\Schedule;
 
class BlishSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('blish:process')
            ->hourly();
    }
}
```

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

```php
// Get stats for a campaign
$stats = $campaign->stats;
// $stats->opens, $stats->clicks, $stats->sent
```

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
```

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
