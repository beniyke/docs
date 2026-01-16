# Mail

The Mail system provides a simple interface for sending emails in your application using the `Mailable` contract and `Mailer` service.

## Configuration

Mail configuration is stored in `App/Config/mail.php`.

```php
return [
    'sender' => ['email' => env('MAIL_SENDER'), 'name' => env('APP_NAME')],
    'paths' => [
        'debug' => 'public/sent-mail',
        'template' => 'App/storage/email-templates',
    ],
    'builder' => [
        'brand' => [
            'name' => env('APP_NAME'),
            'logo' => 'img/logo.png',
        ],
        'templates' => [
            'default' => 'singlepage.html',
            'campaign' => 'campaign.html',
        ],
    ],
    'smtp' => [
        'status' => env('MAIL_SMTP', true),
        'host' => env('MAIL_HOST'),
        'username' => env('MAIL_USERNAME'),
        'password' => env('MAIL_PASSWORD'),
        'port' => env('MAIL_PORT', 587),
        'encryption' => env('MAIL_ENCRYPTION', 'tls'),
        'auth' => true,
    ],
];
```

## Creating Mail Notifications

The easiest way to create a new mail notification is using the CLI generator command.

### Generate the Class

Run the following command to scaffold a new notification class:

```bash
php dock email-notification:create WelcomeEmail Auth
```

This creates a `WelcomeEmail` class inside the `App/Modules/Auth/Notifications/Email` directory (assuming a modular structure).

### Class Structure

The generated class extends `App\Core\BaseEmailNotification`, which handles the underlying `toMail` logic and `EmailBuilder` integration. You only need to implement the following methods to define your content:

- **`getRecipients()`**: Define recipient(s). Supports `to`, `cc`, `bcc`, and `reply_to` keys.
- **`getSubject()`**: Define the email subject.
- **`getTitle()`**: Define the main title (H1) of the email.
- **`getRawMessageContent()`**: Define the body content using `EmailComponent` or raw HTML.

```php
<?php

namespace App\Notifications\Email;

use App\Core\BaseEmailNotification;
use Mail\Core\EmailComponent;

class WelcomeEmail extends BaseEmailNotification
{
    public function getRecipients(): array
    {
        return [
            'to' => [
                $this->payload->get('email') => $this->payload->get('name'),
            ],
        ];
    }

    public function getSubject(): string
    {
        return 'Welcome to Our Platform!';
    }

    public function getTitle(): string
    {
        return 'Welcome!';
    }

    protected function getRawMessageContent(): string
    {
        $name = $this->payload->get('name');

        // Use EmailComponent for fluent HTML generation
        return EmailComponent::make()
            ->greeting("Hello {$name},")
            ->line('Welcome aboard! We are excited to have you.')
            ->action('Login', url('/login'))
            ->render();
    }
}
```

### Notification Options

You can override additional methods to customize the email behavior:

```php
// Define custom sender (defaults to config mail.sender)
public function getSender(): ?array
{
    return ['name' => 'Support Team', 'email' => 'support@example.com'];
}

// Add preheader text (visible in inbox preview)
public function getPreheader(): ?string
{
    return 'Your account has been successfully created.';
}

// Use a different template (defaults to 'default')
public function getTemplate(): string
{
    return 'invoice'; // Must map to a template in mail.php config
}
```

## Sending Emails

There are three ways to send emails. All return a `Mail\MailStatus` object.

### Using the Mail Facade (Recommended)

```php
use Helpers\Data;
use Mail\Mail;
use App\Notifications\Email\WelcomeEmail;

Mail::send(new WelcomeEmail(
    Data::make(['name' => 'John', 'email' => 'john@example.com'])
));
```

### Using the Helper

```php
use Helpers\Data;
use App\Notifications\Email\WelcomeEmail;

mailer(new WelcomeEmail(
    Data::make(['name' => 'John', 'email' => 'john@example.com'])
));
```

### Using the Service

```php
use Helpers\Data;
use Mail\Mailer;
use App\Notifications\Email\WelcomeEmail;

resolve(Mailer::class)->send(new WelcomeEmail(
    Data::make(['name' => 'John', 'email' => 'john@example.com'])
));
```

## Advanced: Customizing with EmailBuilder

The `BaseEmailNotification` class automatically uses the `EmailBuilder` service to render your email content. However, if you need full control, you can override the `toMail` method directly.

### Configure Templates

Ensure your templates exist in the path defined in `mail.paths.template` (default: `App/storage/email-templates`). Map them in `mail.builder.templates`.

**Template Syntax:**
The renderer replaces `[KEY]` with values.

```html
<!-- singlepage.html -->
<h1>[TITLE]</h1>
<img src="[LOGO]" />
<div>[CONTENT]</div>
<footer>[FOOTNOTE]</footer>
```

### Usage

If you choose to override the `toMail` method for full control, you can inject the `EmailBuilder` to render templates ly:

```php
public function toMail(EmailBuilder $builder): Data
{
    // ... custom logic ...
    return Data::make([/* ... */]);
}
```

## Attachments

To add attachments, provide an array of file definitions in the `attachment` key.

```php
    public function getAttachment(): array
    {
        return [
            '/absolute/path/to/invoice.pdf' => 'Invoice.pdf',
            '/absolute/path/to/image.png' => 'Image.png',
        ];
    }
```

## Building Content with EmailComponent

For a fluent, object-oriented way to build your email body, use the `Mail\Core\EmailComponent` class. This is useful for constructing the main content string passed to `EmailBuilder` or `Data`.

```php
use Mail\Core\EmailComponent;

protected function getRawMessageContent(): string
{
    $name = $this->payload->get('name');

    return EmailComponent::make()
        ->greeting("Hello {$name},")
        ->line('Welcome to our platform! We are excited to have you.')
        ->action('Go to Dashboard', url('/dashboard'))
        ->line('If you did not create an account, no further action is required.')
        ->render();
}
```

### Fluent Methods Reference

The `EmailComponent` provides a variety of methods to build structured, responsive emails.

- **`greeting(string $text)`**
  Adds a large H1 greeting header. Use this at the very top of your email.

  ```php
  ->greeting('Hello User,')
  ```

- **`line(string $text)`**
  Adds a standard paragraph of text. This is your primary method for adding content.

  ```php
  ->line('Verified email addresses allow you to recover access...')
  ```

- **`action(string $text, string $url)`**
  Adds a prominent call-to-action button. Use this for the primary goal of the email (e.g., "Reset Password", "View Invoice").

  ```php
  ->action('View Dashboard', url('/dashboard'))
  ```

- **`status(string $message, string $type = 'success')`**
  Adds a colored status banner. Useful for alerts or system messages.
  _Supported types:_ `'success'` (green), `'warning'` (yellow), `'error'` (red).

  ```php
  ->status('Your subscription has been renewed.', 'success')
  ```

- **`hero(string $url, ?string $alt = null)`**
  Adds a full-width, centered hero image. Great for marketing emails or visual announcements.

  ```php
  ->hero('https://example.com/banner.jpg', 'New Feature')
  ```

- **`list(array $items)`**
  Adds a bulleted list of items.

  ```php
  ->list(['Step 1: Sign up', 'Step 2: Verify', 'Step 3: Enjoy'])
  ```

- **`table(array $data)`**
  Adds a key-value table. Perfect for connection details, summaries, or simple invoices.

  ```php
  ->table(['Plan' => 'Pro', 'Amount' => '$10.00'])
  ```

- **`attributes(array $attributes)`**
  Adds a boxed section of metadata properties. Use this for technical details like IP addresses, Device, or Time.

  ```php
  ->attributes(['IP Address' => '127.0.0.1', 'Device' => 'Chrome Windows'])
  ```

- **`panel(string $text)`**
  Adds a highlighted panel block with a left border. Use this to emphasize important notes or instructions.

  ```php
  ->panel('Please keep this code confidential.')
  ```

- **`divider()`**
  Adds a visual separator line to break up sections.

  ```php
  ->divider()
  ```

- **`subcopy(string $text)`**
  Adds small, gray text. Use this for secondary information, disclaimers, or "trouble clicking" help links.

  ```php
  ->subcopy('If you did not request this, please ignore this email.')
  ```

- **`raw(string $html)`**
  Adds raw, unsanitized HTML directly to the output.
  _Warning:_ Use with caution and ensure content is safe/sanitized.

  ```php
  ->raw('<strong>Custom HTML</strong>')
  ```

- **`render()`**
  Finalizes the component and returns the generated HTML string.

## Debugging & Development

When your application is in debug mode (`APP_DEBUG=true`) or `mail.smtp.debug` is enabled, the framework automatically saves a copy of every generated email to your public directory for inspection.

- **Location**: `public/sent-mail` (configurable via `mail.paths.debug`)
- **File Name**: `mail-{timestamp}.html`

This allows you to visually verify email layouts and content during local development without sending actual emails.

### Sending Mail

To send an email, use the `Mail::send` method. It accepts a `Mailable` instance.

```php
use Mail\Mail;
use App\Notifications\Email\WelcomeEmail;
use Helpers\Data;

$payload = Data::make(['name' => 'John', 'email' => 'john@example.com']);
$status = Mail::send(new WelcomeEmail($payload));

if ($status->isSuccessful()) {
    echo "Sent!";
} else {
    echo "Error: " . $status->getMessage();
}
```

The `send` method returns a `Mail\MailStatus` object which provides the following methods:

- `isSuccessful(): bool`: Returns true if sent successfully.
- `isFailed(): bool`: Returns true if sending failed.
- `getMessage(): string`: Returns the success or error message.

## Deferred Sending

To prevent email sending logic (which might involve template rendering or database operations) from slowing down your HTTP response, you can use `Mail::deferred()`. This method queues the email sending task to be executed _after_ the response is sent to the browser.

```php
use Mail\Mail;
use App\Notifications\Email\WelcomeEmail;
use Helpers\Data;

// Email sending will be handled after the response
$payload = Data::make(['name' => 'John', 'email' => 'john@example.com']);
Mail::deferred(new WelcomeEmail($payload));
```
