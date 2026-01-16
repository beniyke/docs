# Verify

The Verify package provides production-ready OTP (One-Time Password) verification for implementing 2FA (two-factor authentication) in your application. It supports multiple delivery channels (email, SMS, etc.) with built-in rate limiting, security logging, and comprehensive error handling.

## Features

- **Channel-Agnostic**: Send OTPs via email, SMS, or custom channels
- **Cryptographically Secure**: Uses `random_int()` for code generation
- **Rate Limiting**: Prevents brute-force and abuse attacks
- **Password Hashing**: Codes stored securely using bcrypt
- **Automatic Expiration**: Time-bound codes (configurable)
- **Model Integration**: Trait for elegant usage with Eloquent models
- **Audit Logging**: Security events logged for compliance
- **GDPR Compliant**: Identifier masking in logs

## Prerequisites

- **PHP 8.2+**
- **Database**: Migration support for OTP storage
- **Notify Package**: Required for default Email and SMS channels

## Installation

Verify is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Verify --packages
```

This will automatically:

- Run database migrations (`verify_*` tables)
- Register the service provider
- Discover console commands

### Configuration

Configuration file: `App/Config/verify.php`

```php
return [
    // OTP code length (4-8 digits)
    'code_length' => 6,

    // Code expiration in minutes
    'expiration_minutes' => 15,

    // Rate limiting for code generation
    'rate_limit_generation' => 3,  // Max 3 codes per hour
    'rate_limit_window_minutes' => 60,

    // Verification attempt limits
    'max_verification_attempts' => 5,

    // Available delivery channels
    'channels' => [
        'email' => Verify\Channels\EmailChannel::class,
        'sms' => Verify\Channels\SmsChannel::class,
    ],

    // Default channel
    'default_channel' => 'email',
];
```

Environment variables (`.env`):

```env
VERIFY_CODE_LENGTH=6
VERIFY_EXPIRATION_MINUTES=15
VERIFY_RATE_LIMIT_GENERATION=3
VERIFY_RATE_LIMIT_WINDOW=60
VERIFY_MAX_ATTEMPTS=5
VERIFY_DEFAULT_CHANNEL=email
```

## Basic Usage

### Static Facade

```php
use Verify\Verify;

// Generate and send OTP
$code = Verify::send('user@example.com', 'email');
// Returns: true (code generated and sent via email)

// Generate code without sending
$code = Verify::generate('user@example.com', 'email');
// Returns: "123456"

// Check if user has a pending OTP
if (Verify::hasPending('user@example.com')) {
    // User already has a pending OTP - maybe show "Resend" instead
}

// Verify OTP
$isValid = Verify::verify('user@example.com', '123456');
// Returns: true

// Resend OTP
Verify::resend('user@example.com', 'email');

// Delete OTP
Verify::delete('user@example.com');
```

## Model Integration (Recommended)

Add the `HasOtpVerification` trait to any model:

```php
use Verify\Traits\HasOtpVerification;

class User extends Model
{
    use HasOtpVerification;
}
```

### Usage with Models

```php
// Find user
$user = User::query()->where('email', 'john@example.com')->first();

// Send OTP to user's email
$code = $user->sendOtp(); // Uses default channel (email)

// Or explicitly specify channel
$code = $user->sendOtp('sms'); // Send via SMS

// In controller - verify OTP entered by user
public function verify(): Response
{
    $user = User::query()->where('email', $this->request->post('email'))->first();

    if ($user->verifyOtp($this->request->post('code'))) {
        // Code is valid - log user in
        $this->request->session()->set('user_id', $user->id);
        $this->request->session()->set('2fa_verified', true);

        return $this->response->redirect($this->request->fullRouteByName('dashboard'));
    } else {
        // Code is invalid
        $this->flash->error('Invalid code');
        return $this->response->back();
    }
}

// Resend OTP
$user->resendOtp();

// Check if user has pending OTP
if ($user->hasPendingOtp()) {
    // Show "Resend OTP" button instead of "Send OTP"
}

// Delete OTP (logout, cancel, etc.)
$user->deleteOtp();
```

### Custom Identifier Logic

Override `getOtpIdentifier()` to customize how the identifier is determined:

```php
class User extends Model
{
    use HasOtpVerification;

    protected function getOtpIdentifier(?string $channel = null): string
    {
        // Use phone for SMS with country code
        if ($channel === 'sms') {
            return $this->country_code . $this->phone;
        }

        // Use backup email if primary is unverified
        if (!$this->email_verified_at && $this->backup_email) {
            return $this->backup_email;
        }

        // Default to email
        return $this->email;
    }
}
```

## Complete 2FA Implementation

### Registration with Email Verification

```php
use Verify\Exceptions\OtpExpiredException;
use Verify\Exceptions\OtpInvalidException;
use Verify\Exceptions\RateLimitExceededException;

class AuthController extends BaseController
{
    public function register(): Response
    {
        if (!$this->request->isPost()) {
            return $this->response->redirect($this->request->fullRoute());
        }

        // Create user
        $user = new User();
        $user->name = $this->request->post('name');
        $user->email = $this->request->post('email');
        $user->password = password_hash($this->request->post('password'), PASSWORD_DEFAULT);
        $user->save();

        // Send OTP for email verification
        $user->sendOtp('email');

        // Store user ID in session
        $this->request->session()->set('pending_user_id', $user->id);

        return $this->asView('auth/verify-otp');
    }

    public function verifyOtp(): Response
    {
        $userId = $this->request->session()->get('pending_user_id');
        $user = User::find($userId);

        try {
            if ($user->verifyOtp($this->request->post('code'))) {
                // Mark email as verified
                $user->email_verified_at = date('Y-m-d H:i:s');
                $user->save();

                // Log user in
                $this->request->session()->set('user_id', $user->id);
                $this->request->session()->delete('pending_user_id');

                return $this->response->redirect($this->request->fullRouteByName('dashboard'));
            }
        } catch (OtpExpiredException $e) {
            $this->flash->error('Code has expired. Request a new one.');
            return $this->response->back();
        } catch (OtpInvalidException $e) {
            $this->flash->error('Invalid code. Please try again.');
            return $this->response->back();
        } catch (RateLimitExceededException $e) {
            $this->flash->error('Too many attempts. Please try again later.');
            return $this->response->back();
        }

        $this->flash->error('Verification failed');
        return $this->response->back();
    }

    public function resendOtp(): Response
    {
        $userId = $this->request->session()->get('pending_user_id');
        $user = User::find($userId);

        try {
            $user->resendOtp();
            $this->flash->success('Code resent successfully');
            return $this->response->back();
        } catch (RateLimitExceededException $e) {
            $this->flash->error('Please wait before requesting another code');
            return $this->response->back();
        }
    }
}
```

### Login with 2FA

```php
use Verify\Exceptions\OtpExpiredException;
use Verify\Exceptions\OtpInvalidException;
use Verify\Exceptions\RateLimitExceededException;

class LoginController extends BaseController
{
    public function login(): Response
    {
        if (!$this->request->isPost()) {
            return $this->response->redirect($this->request->fullRoute());
        }

        $user = User::query()->where('email', $this->request->post('email'))->first();

        if (!$user || !password_verify($this->request->post('password'), $user->password)) {
            $this->flash->error('Invalid credentials');
            return $this->response->back();
        }

        // Send 2FA code
        $user->sendOtp();

        $this->request->session()->set('2fa_user_id', $user->id);

        return $this->asView('auth/2fa-verify');
    }

    public function verify2fa(): Response
    {
        $userId = $this->request->session()->get('2fa_user_id');
        $user = User::find($userId);

        try {
            if ($user->verifyOtp($this->request->post('code'))) {
                // 2FA successful
                $this->request->session()->set('user_id', $user->id);
                $this->request->session()->delete('2fa_user_id');

                return $this->response->redirect($this->request->fullRouteByName('dashboard'));
            }
        } catch (OtpExpiredException $e) {
            $this->flash->error('Code expired');
            return $this->response->back();
        } catch (OtpInvalidException $e) {
            $this->flash->error('Invalid code');
            return $this->response->back();
        } catch (RateLimitExceededException $e) {
            $this->flash->error('Too many attempts');
            return $this->response->back();
        }

        $this->flash->error('Verification failed');
        return $this->response->back();
    }
}
```

## Exception Handling

The package throws specific exceptions for different failure scenarios:

```php
use Verify\Exceptions\OtpExpiredException;
use Verify\Exceptions\OtpInvalidException;
use Verify\Exceptions\OtpNotFoundException;
use Verify\Exceptions\RateLimitExceededException;
use Verify\Exceptions\ChannelNotFoundException;

try {
    Verify::verify('user@example.com', '123456');
} catch (OtpNotFoundException $e) {
    // No OTP exists for this identifier
    // Action: Ask user to request a code
} catch (OtpExpiredException $e) {
    // OTP has expired (older than expiration_minutes)
    // Action: Ask user to request a new code
} catch (OtpInvalidException $e) {
    // Wrong code entered
    // Action: Show error, allow retry
} catch (RateLimitExceededException $e) {
    // Too many attempts
    // Action: Block further attempts, show cooldown message
} catch (ChannelNotFoundException $e) {
    // Invalid channel specified
    // Action: Log error, use default channel
}
```

## Service Injection

For advanced use cases, inject the `VerifyManager`:

```php
use Verify\Services\VerifyManager;

class CustomAuthService
{
    public function __construct(
        private readonly VerifyManager $verify
    ) {}

    public function sendCode(string $identifier, string $channel = 'email'): string
    {
        return $this->verify->generate($identifier, $channel);
    }

    public function validateCode(string $identifier, string $code): bool
    {
        try {
            return $this->verify->verify($identifier, $code);
        } catch (\Exception $e) {
            logger('verify.log')->error('Verification failed', [
                'identifier' => $identifier,
                'error' => $e->getMessage(),
            ]);
            return false;
        }
    }
}
```

## Console Commands

### Cleanup Expired Codes

Removes expired OTP codes and old attempt records:

```bash
php dock verify:cleanup

# With custom retention period
php dock verify:cleanup --days=30
```

**Schedule in Cron** (recommended):

```bash
# Run cleanup daily at 2 AM
0 2 * * * cd /path/to/app && php dock verify:cleanup
```

### View Statistics

Display OTP usage statistics and metrics:

```bash
php dock verify:stats
```

Output:

```
OTP Verification Statistics
============================

Overview
┌─────────────────────────┬───────┐
│ Metric                  │ Value │
├─────────────────────────┼───────┤
│ Active Codes            │ 12    │
│ Generated Today         │ 156   │
│ Verified Today          │ 142   │
│ Success Rate Today      │ 91.03%│
│ Expired Codes           │ 8     │
│ Rate Limit Violations   │ 3     │
└─────────────────────────┴───────┘

Channel Usage (Today)
┌─────────┬───────┐
│ Channel │ Count │
├─────────┼───────┤
│ email   │ 152   │
│ sms     │ 4     │
└─────────┴───────┘
```

## Customizing Message Content

The default channels (`EmailChannel` and `SmsChannel`) use the **Notify** package with preset notification templates.

To customize the message content (e.g., change the email subject or SMS body), you should **create a custom channel**.

### Example: Custom Email Channel

1. Create a new notification class (optional, or use your own mail logic):

```php
namespace App\Notifications;

use Mail\EmailNotification;

class CustomOtpEmail extends EmailNotification
{
    // ... custom email logic ...
    public function getSubject(): string {
        return 'Here is your login code';
    }
}
```

2. Create a custom channel that uses your notification:

```php
namespace App\Channels;

use Verify\Contracts\ChannelInterface;
use Notify\Notify;
use App\Notifications\CustomOtpEmail;

class CustomEmailChannel implements ChannelInterface
{
    public function send(string $identifier, string $code, ?string $receiverName = null): bool
    {
        // Use Notify to send your custom notification
        $result = Notify::email(CustomOtpEmail::class, [
            'email' => $identifier,
            'code' => $code
        ]);

        return $result['status'] === 'success';
    }
}
```

3. Register it in `verify.php` config:

```php
'channels' => [
    'email' => App\Channels\CustomEmailChannel::class, // Override default 'email'
],
```

## Custom Channels

Create custom delivery channels by implementing `ChannelInterface`:

```php
namespace Verify\Channels;

use Verify\Contracts\ChannelInterface;

class WhatsAppChannel implements ChannelInterface
{
    public function send(string $identifier, string $code): bool
    {
        // Implement WhatsApp API integration
        $client = new WhatsAppClient(config('whatsapp.api_key'));

        $message = "Your verification code is: {$code}. Valid for 15 minutes.";

        try {
            $client->sendMessage($identifier, $message);

            logger('verify.log')->info('OTP sent via WhatsApp', [
                'to' => $identifier,
            ]);

            return true;
        } catch (\Exception $e) {
            logger('verify.log')->error('WhatsApp send failed', [
                'to' => $identifier,
                'error' => $e->getMessage(),
            ]);

            return false;
        }
    }
}
```

Register in `Config/verify.php`:

```php
'channels' => [
    'email' => EmailChannel::class,
    'sms' => SmsChannel::class,
    'whatsapp' => WhatsAppChannel::class, // New channel
],
```

Use the custom channel:

```php
Verify::generate('1234567890', 'whatsapp');
// or
$user->sendOtp('whatsapp');
```

## Security Best Practices

- **Always Use HTTPS**: OTP codes should only be transmitted over secure connections
- **Rate Limiting**: The package includes built-in rate limiting, but consider additional application-level limits
- **Logging**: All OTP operations are logged to `App/storage/logs/verify.log` for security audits
- **Identifier Privacy**: Identifiers are automatically masked in logs (GDPR compliant)
- **Code Complexity**: Use 6-digit codes minimum for production
- **Expiration**: Keep expiration short (15 minutes recommended)
- **Single Use**: Codes are automatically invalidated after successful verification

## Troubleshooting

### Codes not being sent

**Check**:

- Mail configuration is correct (`App/Config/mail.php`)
- Email channel is properly configured
- Check `verify.log` for errors

### Rate limit errors

**Solution**:

- Adjust `rate_limit_generation` and `rate_limit_window_minutes` in config
- Use `verify:stats` to monitor violations
- Clear attempts: `DELETE FROM verify_attempts WHERE identifier = 'user@example.com'`

### Codes always invalid

**Check**:

- Identifier normalization (should be lowercase)
- Check for typing errors (O vs 0, l vs 1)

## Analytics

The `Verify` package includes an analytics service to monitor OTP success rates and channel performance.

```php
use Verify\Analytics;

// Get overall success metrics
$stats = Analytics::getSuccessMetrics($from, $to);
// Returns: total_generated, total_verified, total_expired, success_rate

// Get daily volume of generation vs verification
$volume = Analytics::getDailyVolume($from, $to);

// Compare success rates across different channels
$channels = Analytics::getChannelStats();

// Monitor rate-limit activity
$limits = Analytics::getRateLimitStats();
```

You can also access it via the main `Verify` facade:

```php
Verify::analytics()->getSuccessMetrics();
```

## Service API Reference

### Verify (Facade)

| Method                                            | Description                                                     |
| :------------------------------------------------ | :-------------------------------------------------------------- |
| `send(string $id, string $chan, ?string $name)`   | Generates and transmits a secure OTP via the specified channel. |
| `verify(string $id, string $code)`                | Validates an OTP code with rate limiting protection.            |
| `resend(string $id, string $chan, ?string $name)` | Rebottles the existing valid OTP to the receiver.               |
| `delete(string $id)`                              | Manually purges OTP records for an identifier.                  |
| `hasPending(string $id)`                          | Checks if there is currently an unexpired OTP for the ID.       |
| `analytics()`                                     | Returns the `VerifyAnalytics` service instance.                 |

### Analytics (Facade)

| Method                            | Description                                                 |
| :-------------------------------- | :---------------------------------------------------------- |
| `getSuccessMetrics(?$from, ?$to)` | Returns overall verification metrics and success rate.      |
| `getDailyVolume(string $f, $t)`   | Returns daily volume of generation vs verification.         |
| `getChannelStats()`               | Returns success rates grouped by channel (email, sms, etc). |
| `getRateLimitStats()`             | Returns activity counts for rate-limited identifiers.       |

### VerifyManager

The `VerifyManager` is the underlying service class. You can inject it directly if you prefer dependency injection over facades. It provides the same methods as the facade.

| Method                                            | Description                                 |
| :------------------------------------------------ | :------------------------------------------ |
| `generate(string $id, string $chan)`              | Generates a code without sending it.        |
| `send(string $id, string $chan, ?string $name)`   | Generates and sends a code.                 |
| `verify(string $id, string $code)`                | Verifies a code.                            |
| `resend(string $id, string $chan, ?string $name)` | Resends the last valid code.                |
| `delete(string $id)`                              | Deletes pending codes for an identifier.    |
| `hasPending(string $id)`                          | Checks if an identifier has a pending code. |

---

`HasOtpVerification` Trait

```php
$model->sendOtp(?string $channel = null, ?string $identifier = null, ?string $receiverName = null): bool
$model->verifyOtp(string $code, ?string $identifier = null): bool
$model->resendOtp(?string $channel = null, ?string $identifier = null, ?string $receiverName = null): bool
$model->deleteOtp(?string $identifier = null): bool
$model->hasPendingOtp(?string $identifier = null): bool
```

### Configuration Methods

```php
config('verify.code_length')                    // int (4-8)
config('verify.expiration_minutes')             // int
config('verify.rate_limit_generation')          // int
config('verify.rate_limit_window_minutes')      // int
config('verify.max_verification_attempts')      // int
config('verify.channels')                       // array
config('verify.default_channel')                // string
```

## See Also

- [Mail](mail.md) - Email delivery configuration
- [Authentication](authentication.md) - User authentication
- [Security](security.md) - Security best practices
