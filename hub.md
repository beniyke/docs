# Hub

Hub provides team collaboration features: threaded conversations, messaging, @mentions, reactions, and scheduled reminders. It integrates with the Link package for team invites.

## Features

- **Threads**: Create threaded discussions attached to any resource.
- **Messages**: Post messages with reply threading and @mentions.
- **Reactions**: Emoji reactions on messages.
- **Reminders**: Schedule reminders with repeat intervals (daily, weekly, monthly).
- **Invites**: Generate invite links via the Link package.
- **Analytics**: Track engagement, message trends, and contributor activity.

## Installation

Hub is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Hub --packages
```

This will automatically:

- Run database migrations for all Hub tables.
- Register the `HubServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/hub.php`

```php
return [
    'max_message_length' => 10000,
    'max_attachments_per_message' => 10,
    'max_members_per_thread' => 100,
    'default_notifications_enabled' => true,
    'mention_pattern' => '/@(\w+)/',
    'reminder_advance_minutes' => 15,
    'max_snooze_count' => 5,
    'retention_days' => 365,
];
```

## Basic Usage

### Creating Threads

```php
use Hub\Hub;

$user = $this->auth->user();

// Create a thread attached to a project
$thread = Hub::thread()
    ->on($project)
    ->title('Sprint Planning')
    ->members([$user1->id, $user2->id])
    ->by($user->id)
    ->create();
```

### Posting Messages

```php
use Hub\Hub;

// Post a message
$message = Hub::message()
    ->in($thread)
    ->from($user->id)
    ->body('Hey @jane, can you review this?')
    ->send();

// Reply to a message
Hub::message()
    ->replyTo($message)
    ->from($user2->id)
    ->body('Done! ✅')
    ->send();
```

### Reactions

```php
use Hub\Hub;

// Add a reaction
Hub::react($message, '👍', $userId);

// Toggle reaction (removes if exists, adds if not)
Hub::toggleReaction($message, '❤️', $userId);

// Remove a reaction
Hub::removeReaction($message, '👍', $userId);
```

### Reminders

```php
use Hub\Hub;

// Create a reminder for tomorrow
$reminder = Hub::reminder()
    ->for($user->id)
    ->message('Follow up on client proposal')
    ->tomorrow(9, 0)  // 9:00 AM
    ->create();

// Create a weekly recurring reminder
Hub::reminder()
    ->for($userId)
    ->about($project)
    ->message('Weekly standup')
    ->inHours(24)
    ->weekly()
    ->create();

// Snooze a reminder
$reminder->snooze(15);  // 15 minutes

// Complete a reminder
$reminder->complete();
```

### Team Invites (Link Integration)

```php
use Hub\Hub;

// Create an invite link for a thread
$link = Hub::invite($thread, 'newmember@company.com');

// Get the shareable URL
$url = $link->signedUrl();
```

## Model Helpers

### Thread Model

```php
$thread = Hub::find($refid);

$thread->hasMember($userId);      // Check membership
$thread->getMember($userId);      // Get member record
$thread->getMessageCount();       // Total messages
$thread->getUnreadCount($userId); // Unread for user
$thread->pin();                   // Pin thread
$thread->unpin();                 // Unpin thread
```

### Message Model

```php
$message = Hub::findMessage($refid);

$message->isReply();               // Is this a reply?
$message->hasReaction($uid, '👍'); // User reacted?
$message->getReactionCounts();     // {emoji: count}
$message->pin();                   // Pin message
```

### Reminder Model

```php
$reminder = Hub::findReminder($refid);

$reminder->isDue();       // Is reminder due?
$reminder->repeats();     // Does it repeat?
$reminder->snooze(15);    // Snooze 15 minutes
$reminder->complete();    // Mark complete
$reminder->cancel();      // Cancel reminder
```

### Thread Member

```php
$member = $thread->getMember($userId);

$member->isOwner();      // Is owner?
$member->isAdmin();      // Is admin?
$member->canManage();    // Can manage members?
$member->markAsRead();   // Mark thread read
$member->toggleNotifications();  // Toggle notifications
```

## Events

Hub dispatches events for notifications and integrations:

| Event           | Payload                   | Use Case                   |
| --------------- | ------------------------- | -------------------------- |
| `MessagePosted` | message, threadId, userId | Notify thread members      |
| `UserMentioned` | mention, message, userId  | Send @mention notification |
| `ReminderDue`   | reminder, userId          | Send reminder notification |
| `ThreadCreated` | thread, createdBy         | Notify invited members     |

### Example Event Listener

```php
use Hub\Events\UserMentioned;
use Ally\Ally;

class SendMentionNotification
{
    public function handle(UserMentioned $event): void
    {
        Ally::send(
            User::find($event->mentionedUserId),
            new MentionNotification($event->message)
        );
    }
}
```

## Analytics

```php
use Hub\Hub;

$analytics = Hub::analytics();

// Active threads with message counts
$active = $analytics->getActiveThreads(30);

// Daily message volume
$trends = $analytics->getMessageTrends(30);

// Engagement metrics
$metrics = $analytics->getEngagementMetrics();
// Returns: total_threads, total_messages, active_users, avg_messages_per_thread

// Top contributors
$contributors = $analytics->getTopContributors(10);

// Reaction distribution
$reactions = $analytics->getReactionDistribution();

// Reminder metrics
$reminders = $analytics->getReminderMetrics();
// Returns: total, completed, pending, snoozed, completion_rate
```

## Package Integrations

### Link Package

Hub uses Link for team invites:

```php
$link = Hub::invite($thread, 'user@example.com');
```

When the invite is used (validated via `Link::validate()`), add the user to the thread:

```php
$link = Link::validate($token);
if ($link->canJoin()) {
    $link->linkable
        ->addMember($newUserId)
        ->asGuest()
        ->add();
}
```

### Support Package

Attach threads to support tickets for internal team discussions:

```php
use Hub\Hub;
use Support\Support;

// Create a thread on a support ticket
$ticket = Support::findTicket($ticketId);

$thread = Hub::thread()
    ->on($ticket)
    ->title("Internal: {$ticket->subject}")
    ->members([$agent1->id, $agent2->id])
    ->by($user->id)
    ->create();
```

### Slot Package

Add discussion threads to bookings for coordination:

```php
use Hub\Hub;
use Slot\Slot;

// Create a thread for a booking
$booking = Slot::booking($bookingId);

$thread = Hub::thread()
    ->on($booking)
    ->title("Booking Discussion: {$booking->schedule->name}")
    ->members([$booking->user_id, $booking->host_id])
    ->create();
```

### Mail Notifications

Hub sends notifications using `Mail::send()` with built-in notification classes:

```php
use Helpers\Data;
use Hub\Events\UserMentioned;
use Hub\Notifications\MentionNotification;
use Mail\Mail;

class SendMentionNotification
{
    public function handle(UserMentioned $event): void
    {
        $user = User::find($event->mentionedUserId);

        Mail::send(new MentionNotification(Data::make([
            'email' => $user->email,
            'mentioner_name' => $event->message->user->name,
            'thread_title' => $event->message->thread->title,
            'message_preview' => Str::limit($event->message->body, 100),
        ])));
    }
}
```

**Available notifications:**

- `MentionNotification` - For @mentions
- `ReminderNotification` - For due reminders
- `MessagePostedNotification` - For new messages

### Audit Package

Log thread and message activity:

```php
use Audit\Audit;

Audit::make()
    ->event('hub.message.posted')
    ->on($message)
    ->by($userId)
    ->log();
```

## Service API Reference

### Hub (Facade)

| Method                             | Description               |
| ---------------------------------- | ------------------------- |
| `thread()`                         | Returns `ThreadBuilder`   |
| `message()`                        | Returns `MessageBuilder`  |
| `reminder()`                       | Returns `ReminderBuilder` |
| `react($message, $emoji, $userId)` | Add reaction              |
| `invite($thread, $email)`          | Create invite via Link    |
| `find($refid)`                     | Find thread by refid      |
| `findMessage($refid)`              | Find message by refid     |
| `findReminder($refid)`             | Find reminder by refid    |
| `analytics()`                      | Returns `HubAnalytics`    |

### ThreadBuilder

| Method            | Description        |
| ----------------- | ------------------ |
| `on($model)`      | Attach to resource |
| `title($title)`   | Set thread title   |
| `members([$ids])` | Set member IDs     |
| `addMember($id)`  | Add single member  |
| `pinned()`        | Pin the thread     |
| `by($userId)`     | Set creator        |
| `create()`        | Create thread      |

### MessageBuilder

| Method              | Description           |
| ------------------- | --------------------- |
| `in($thread)`       | Set target thread     |
| `replyTo($message)` | Reply to message      |
| `from($userId)`     | Set sender            |
| `body($text)`       | Set message body      |
| `mentions([$ids])`  | Set explicit mentions |
| `pinned()`          | Pin message           |
| `send()`            | Post message          |

### ReminderBuilder

| Method               | Description            |
| -------------------- | ---------------------- |
| `for($userId)`       | Set reminder recipient |
| `about($model)`      | Attach to resource     |
| `message($text)`     | Set reminder text      |
| `at($datetime)`      | Set specific time      |
| `inMinutes(n)`       | Set relative time      |
| `inHours(n)`         | Set relative time      |
| `tomorrow(h, m)`     | Set tomorrow at time   |
| `daily() / weekly()` | Set repeat interval    |
| `create()`           | Create reminder        |

## Processing Reminders

Run the reminder processor via command:

```bash
php dock hub:remind
```

Schedule this command via cron (every minute recommended):

```bash
* * * * * cd /path/to/project && php dock hub:remind >> /dev/null 2>&1
```
