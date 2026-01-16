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
- Create necessary database tables (`pulse_*`).
- Register the `PulseServiceProvider`.

## Facade API

### Channel Management

```php
use Pulse\Pulse;

// Create a new channel
$channel = Pulse::channel()
    ->name('General Discussion')
    ->description('Standard forum channel')
    ->create();
```

### Thread Management

```php
// Create a new thread
$thread = Pulse::thread()
    ->by($user)
    ->in($channel)
    ->title('Welcome to Pulse')
    ->content('This is the first post in the thread.')
    ->pinned()
    ->create();

// Lock or Pin
Pulse::lock($thread);
Pulse::pin($thread);
```

### Post Management

```php
// Submit a reply
$post = Pulse::post()
    ->by($user)
    ->on($thread)
    ->content('I agree with this!')
    ->create();

// React to a post
Pulse::react($user, $post, '🔥');
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
