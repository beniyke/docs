# Notify

Anchor provides a robust notification system built on a modular **Channel-Adapter** architecture, allowing you to send notifications via Email, In-App, SMS, WhatsApp, and any custom channel you define.

## Installation

The Notify system is provided as an installable system package. Run the following command to install it:

```bash
php dock package:install Notify --system
```

This command performs two key actions:

- **Registers the System Provider**: `Notify\Providers\NotifyServiceProvider` is registered to provide default channels.
- **Publishes Configuration**: Publishes the package configuration files.

## Architecture

The system uses a flexible pattern to decouple the _channel_ (orchestration) from the _adapter_ (transport logic).

- **Manager**: The central hub that dispatches notifications to the correct channel.
- **Channel**: Orchestrates the sending process. It receives the notification and delegates the actual transport to an _Adapter_.
- **Adapter**: The concrete implementation that talks to external APIs (e.g., Twilio, Mailgun, Slack).

This means you can switch your SMS provider from Twilio to MessageBird simply by swapping the **Adapter**, without touching your Channel or Notification classes.

## Implementation

**Default Channels (SMS & WhatsApp)**

The installation scaffolds `SmsAdapter` and `WhatsAppAdapter` in `App/Channels/Adapters`. These are intentionally left empty (returning `true`) so you can implement your specific provider logic.

**Example: Implementing Twilio for SMS**

Open `App/Channels/Adapters/SmsAdapter.php` and implement your sending logic:

```php
<?php

namespace App\Channels\Adapters;

use App\Channels\Adapters\Interfaces\SmsAdapterInterface;
use Notify\Contracts\MessageNotifiable;

class SmsAdapter implements SmsAdapterInterface
{
    public function handle(MessageNotifiable $notification): mixed
    {
        // 1. Get the message payload
        $payload = $notification->toMessage();

        // 2. Extract Data
        $phone = $payload['recipient'];
        $message = $payload['message'];

        // 3. Send using your preferred provider (e.g., Twilio)
        // Twilio::messages->create($phone, ['from' => '...', 'body' => $message]);

        // 4. Return result (logged by the system)
        return ['status' => 'success', 'sid' => '...'];
    }
}
```

## Creating Custom Channels

To add a completely new channel (e.g., **Slack**), follow this pattern to ensure your application remains modular and testable.

### Create the Adapter

**Step 1**

Create an interface and a concrete adapter for your channel logic. It is recommended to extend `MessageAdapterInterface` for consistency if your channel sends messages.

**Interface:** `App/Channels/Adapters/Interfaces/SlackAdapterInterface.php`

```php
namespace App\Channels\Adapters\Interfaces;

use App\Channels\Adapters\Interfaces\MessageAdapterInterface;

interface SlackAdapterInterface extends MessageAdapterInterface
{
}
```

**Implementation:** `App/Channels/Adapters/SlackAdapter.php`

```php
namespace App\Channels;

use App\Channels\Adapters\Interfaces\SlackAdapterInterface;
use Notify\Contracts\MessageNotifiable;

class SlackAdapter implements SlackAdapterInterface
{
    public function handle(MessageNotifiable $notification): mixed
    {
        // Logic to send payload to Slack Webhook
        $payload = $notification->toMessage();

        return ['status' => 'sent'];
    }
}
```

### Create the Channel

**Step 2**

The Channel class injects the Adapter and handles the `send` method called by the Manager.

**File:** `App/Channels/SlackChannel.php`

```php
namespace App\Channels;

use App\Channels\Adapters\Interfaces\SlackAdapterInterface;
use Notify\Contracts\Channel;
use Notify\Contracts\Notifiable;

class SlackChannel implements Channel
{
    private SlackAdapterInterface $adapter;

    public function __construct(SlackAdapterInterface $adapter)
    {
        $this->adapter = $adapter;
    }

    public function send(Notifiable $notification): mixed
    {
        return $this->adapter->handle($notification);
    }
}
```

### Register the Channel

**Step 3**

To register your new channel, it is best practice to create a dedicated Service Provider. References to the system `NotifyServiceProvider` can be used as a guide, but you should create your own in the `App` namespace.

**1. Create Provider:** `App/Providers/NotificationServiceProvider.php`

```php
namespace App\Providers;

use Core\Services\ServiceProvider;
use Notify\NotificationManager;
use App\Channels\SlackChannel;
use App\Channels\Adapters\SlackAdapter;
use App\Channels\Adapters\Interfaces\SlackAdapterInterface;

class NotificationServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // 1. Bind the Adapter Interface to your Concrete Implementation
        $this->container->singleton(SlackAdapterInterface::class, SlackAdapter::class);
    }

    public function boot(): void
    {
        // 2. Register the Channel with the Manager
        $manager = $this->container->get(NotificationManager::class);
        $channel = $this->container->make(SlackChannel::class);

        $manager->registerChannel('slack', $channel);
    }
}
```

**2. Register Provider:** Add your new provider to `App/Config/providers.php`:

```php
return [
    // ...
    App\Providers\NotificationServiceProvider::class,
];
```

## Sending Notifications

Once your channels are registered (default or custom), you can send notifications easily.

### Using the Facade (Recommended)

The `Notify` facade uses magic methods to forward calls to the registered channel name.

```php
use Notify\Notify;

// Send to default channels
Notify::email(WelcomeEmail::class, $payload);
Notify::sms(OtpMessage::class, $payload);

// Send to your CUSTOM channel
Notify::slack(NewOrderAlert::class, $payload);
```

### Using the Helper

```php
notify('slack')->with(NewOrderAlert::class, $payload)->send();
```

### Generating Classes via CLI

You can generate notification classes using the CLI:

```bash
# Email Notification (Module Required)
php dock email-notification:create <Name> <Module>

# Message Notification (Module Optional)
php dock message-notification:create <Name> [<Module>]

# In-App Notification (Module Optional)
php dock inapp-notification:create <Name> [<Module>]
```

Examples:

```bash
php dock email-notification:create WelcomeEmail Auth
php dock message-notification:create OtpMessage Auth
php dock inapp-notification:create NewLoginAlert Auth
```

### Manual Creation

#### Email Notifications

Extend `App\Core\BaseEmailNotification` to handle email logic automatically.

```php
namespace App\Notifications;

use App\Core\BaseEmailNotification;
use Mail\Core\EmailComponent;

class WelcomeEmail extends BaseEmailNotification
{
    public function getRecipients(): array
    {
        return ['to' => [$this->payload->get('email') => $this->payload->get('name')]];
    }

    public function getSubject(): string
    {
        return 'Welcome!';
    }

    public function getTitle(): string
    {
        return 'Welcome to Anchor';
    }

    protected function getRawMessageContent(): string
    {
        return EmailComponent::make()
            ->greeting("Hello {$this->payload->get('name')}")
            ->line('Welcome to the platform!')
            ->render();
    }
}
```

### Non-email Notifications

For non-email channels (Sms, WhatsApp etc), you typically implement a simple method to return the payload.

```php
namespace App\Notifications;

use App\Core\BaseMessageNotification;

class NewLoginAlert extends BaseMessageNotification
{
    // ... define toMessage method ...
}
```
