# Pulse

Pulse is a full-featured discussion forum package for the Anchor Framework. It provides hierarchical channels, real-time threads, moderation, user reputation, and engagement features.

## Features

- **Channels**: Nested categories for organizing discussions.
- **Threads**: Topic-based conversations with view tracking.
- **Posts**: Rich content replies with support for threading.
- **Moderation**: Reporting system, pinning, and locking threads.
- **Engagement**: Reward users with reputation points and badges.
- **Reactions**: Emoji expressions for posts.
- **Subscriptions**: Follow threads for updates.

## Installation

Pulse is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Pulse --packages
```

This command will:

- Publish the `pulse.php` configuration file.
- Run the migration for Pulse tables.
- Register the `PulseServiceProvider`.

## Facade API

### Channel Management

```php
use Pulse\Pulse;

// Create a main channel
$commercial = Pulse::channel()
    ->name('Commercial')
    ->description('Business and sales discussions')
    ->create();

// Create a nested sub-channel
$sales = Pulse::channel()
    ->name('Sales Performance')
    ->parent($commercial->id)
    ->private() // Makes it accessible only to specific groups
    ->order(1)
    ->create();
```

### Thread Management

```php
use Pulse\Pulse;

// Create a new thread with an initial post content
$thread = Pulse::thread()
    ->by($user)
    ->in($channel)
    ->title('Q1 Growth Strategy')
    ->content('Let\'s discuss our roadmap for the first quarter.')
    ->pinned() // Pins to the top of the channel
    ->locked() // Prevents new replies
    ->create();

// Actions on existing threads
Pulse::lock($thread);
Pulse::pin($thread);
```

### Post Management

```php
use Pulse\Pulse;

// standard reply
$post = Pulse::post()
    ->by($user)
    ->on($thread)
    ->content('I have some ideas regarding the expansion.')
    ->create();

// Threaded reply (nested)
Pulse::post()
    ->by($manager)
    ->on($thread)
    ->replyingTo($post->id)
    ->content('Please elaborate on the expansion plan.')
    ->create();

// React to a post
Pulse::react($user, $post, '🔥');
```

## Moderation

Pulse provides a robust moderation system via the `Pulse` facade (proxied to `ModerationManagerService`).

### Reporting Content

Users can report threads or posts for review.

```php
use Pulse\Pulse;

// Report a post
Pulse::report($user, $post, 'Inappropriate content');

// Report a thread
Pulse::report($user, $thread, 'Spam');

// Block a post
Pulse::block($post);

// Unblock content
Pulse::unblock($post);
```

## Engagement & Reputation

Reward users for their participation with reputation points and badges.

```php
use Pulse\Pulse;

// Award reputation points manually
Pulse::awardPoints($user, 10);

// Award a badge
$badge = Pulse::findBadge('top-contributor');
Pulse::awardBadge($user, $badge);
```

## Subscriptions

Users can follow threads to receive notifications for new activity.

```php
use Pulse\Pulse;

// Subscribe a user to a thread
Pulse::subscribe($user, $thread);

// Unsubscribe
Pulse::unsubscribe($user, $thread);

// Get all threads the user is following
$threads = Pulse::getSubscribedThreads($user);
```

## Analytics

```php
$analytics = Pulse::analytics();

$total = $analytics->totalPosts(); // Returns: (int) 1500

// Returns: [['date' => '2026-01-01', 'count' => 50], ...]
$activity = $analytics->dailyActivity();

// Returns: [[Thread Model], [Thread Model], ...]
$popular = $analytics->popularThreads(10);
```

## Configuration

Publish the config to `App/Config/pulse.php` to customize:

- Reputation points awarded for various actions.
- Automatic moderation thresholds.
- Pagination and display settings.

## Integrations

- **Hub**: Notifies users of replies and mentions.
- **Media**: Handles attachments for posts and channel icons.
- **Audit**: Logs moderation actions and thread lifecycle events.
- **Permit**: Manages moderator and administrator permissions.
