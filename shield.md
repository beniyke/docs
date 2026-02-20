# Shield

The Shield package provides a comprehensive CAPTCHA integration layer, securing your Anchor application against automated abuse. It offers a unified, fluent API for multiple providers (Google reCAPTCHA, Cloudflare Turnstile) and includes built-in analytics tracking via the **Audit** package.

## Features

- **Unified API**: Switch between reCAPTCHA and Turnstile without changing application code.
- **Fluent & Static Access**: Use the `Shield` facade (`Shield::verify()`) for backend validation.
- **Frontend Helpers**: Helper functions and ViewModel traits for effortless integration.
- **Analytics Integration**: Automatically logs verification success, failure, and errors using the **Audit** package.
- **Strictly Typed**: Built with strict typing and robust error handling for production stability.

## Installation

Shield is a **package** that requires installation.

### Install the Package

```bash
php dock package:install Shield --packages
```

This will automatically:

- Register the `ShieldServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/shield.php`

Define your driver credentials in your `.env` file:

```env
# Default Driver (recaptcha or turnstile)
SHIELD_DRIVER=recaptcha

# Google reCAPTCHA
RECAPTCHA_SITE_KEY=your-site-key
RECAPTCHA_SECRET=your-secret-key

# Cloudflare Turnstile
TURNSTILE_SITE_KEY=your-site-key
TURNSTILE_SECRET=your-secret-key
```

## Basic Usage

### Frontend Integration

Shield provides flexible ways to render the CAPTCHA widget.

#### Option 1: ViewModel Integration (Recommended)

The cleanest approach is to use the `HasShield` trait in your ViewModel. This keeps your views logic-free.

```php
namespace App\Auth\Views\Models;

use Shield\Traits\HasShield;

class LoginViewModel
{
    use HasShield;
}
```

**In your View Template:**

```php
<!-- 1. Render Script (Head or Footer) -->
<?php echo $model->captchaScript(); ?>

<form method="POST" action="/login">
    <!-- 2. Render Widget inside form -->
    <?php echo $model->captchaWidget(); ?>

    <button type="submit">Login</button>
</form>
```

#### Option 2: Global Helper

For simple views without a ViewModel, use the global `shield()` helper. You can optionally pass a driver name to override the default.

```php
<!-- Render both script and widget (accepts attributes) -->
<?php echo shield()->all(['theme' => 'dark']); ?>

<!-- OR separate them -->
<?php echo shield()->script(); ?>
<?php echo shield('turnstile')->render(['size' => 'compact']); ?>
```

### Backend Validation

Use the `Shield` facade to verify the token. The facade proxies to `ShieldManager`, handling the specific driver logic.

```php
use Shield\Shield;

public function login(Request $request)
{
    // Retrieve token (key depends on provider, e.g. 'g-recaptcha-response')
    $token = $this->request->post('g-recaptcha-response')
          ?? $this->request->post('cf-turnstile-response');

    if (! Shield::verify($token, $this->request->ip())) {
        $this->flash->error('Security check failed. Please try again.');
        return $this->response->redirect($this->request->fullRoute());
    }

    // Proceed with authentication...
}
```

## Use Case Walkthrough

### Analytics & Monitoring

Shield integrates natively with the **Audit** package to track security health. Every verification attempt is logged, including the requester's IP address.

> The **Audit** package must be installed and configured for analytics to function.

**Monitoring Events:**

You can access built-in analytics via the `Shield::analytics()` helper, which aggregates Audit logs into useful metrics. All methods support filtering by `driver` and `dateRange`.

```php
use Shield\Shield;

$analytics = Shield::analytics();

// 1. Get an overview of performance
// Filters: ?string $driver, ?array $dateRange
$overview = $analytics->overview(driver: 'recaptcha');
/**
 * [
 *   'total' => 120,
 *   'verified' => 110,
 *   'failed' => 8,
 *   'errors' => 2,
 *   'success_rate' => 91.67
 * ]
 */

// 2. Get daily trend data for charts
// Filters: int $days, ?string $driver
$trends = $analytics->trends(days: 7);
/**
 * [
 *   ['date' => '2024-01-01', 'verified' => 50, 'failed' => 2, 'errors' => 0, 'total' => 52],
 *   ...
 * ]
 */

// 3. Identify top IP addresses triggering CAPTCHAs
// Filters: int $limit, ?string $driver
$suspiciousIps = $analytics->topIps(limit: 5);
/**
 * [
 *   ['ip' => '192.168.1.1', 'total' => 45, 'failed' => 40, 'fail_rate' => 88.9],
 *   ...
 * ]
 */

// 4. Compare performance across drivers
// Filters: ?array $dateRange
$performance = $analytics->driverPerformance();
/**
 * [
 *   ['driver' => 'recaptcha', 'total' => 100, 'verified' => 90, 'success_rate' => 90.0],
 *   ['driver' => 'turnstile', 'total' => 50, 'verified' => 48, 'success_rate' => 96.0],
 * ]
 */

// 5. Get recent raw logs
// Filters: ?string $driver, int $limit
$logs = $analytics->logs(limit: 10);
```

The events logged to Audit are:

- `captcha.verified`: Validation success.
- `captcha.failed`: Validation rejection (bot detected).
- `captcha.error`: Technical error (e.g., API timeout, bad config).

## Testing

For automated testing, you can swap the `ShieldManagerService` in the container with a mock.

```php
use Shield\Services\ShieldManagerService;

/** @var ShieldManagerService $shield */
$shield = $this->createMock(ShieldManagerService::class);
$shield->method('verify')->willReturn(true);

$this->container->instance(ShieldManagerService::class, $shield);
```

## Service API Reference

### Shield (Facade)

| Method                                 | Description                                                        |
| :------------------------------------- | :----------------------------------------------------------------- |
| `verify(string $token, ?string $ip)`   | Verifies the given token with the default driver. Returns `bool`.  |
| `analytics()`                          | Returns the `ShieldAnalytics` service instance.                    |
| `driver(?string $name)`                | Returns a specific driver instance (e.g. `GoogleRecaptchaDriver`). |
| `render(?string $driver, array $attr)` | Returns the HTML widget code.                                      |
| `getScript(?string $driver)`           | Returns the HTML script tag.                                       |

### Configuration Options

| Option    | Description                                        |
| :-------- | :------------------------------------------------- |
| `default` | The default driver key (`recaptcha`, `turnstile`). |
| `drivers` | Array of driver configurations (keys, secrets).    |

## Troubleshooting

| Issue                         | Cause                                  | Solution                                                  |
| :---------------------------- | :------------------------------------- | :-------------------------------------------------------- |
| **Verification always fails** | IP address mismatch or Invalid Secret. | Check `.env` keys and ensure `request->ip()` is passed.   |
| **Widget not showing**        | JavaScript error or invalid Site Key.  | Check browser console and `.env` site key.                |
| **Connection Timeout**        | Firewall blocking external APIs.       | Ensure server allows outbound HTTPS to Google/Cloudflare. |
