# Refer

The Refer package provides a referral and affiliate system that enables user acquisition tracking, automated rewards, and comprehensive performance analytics.

## Features

- **Automated Rewards**: Flexible reward logic (Fixed, Percentage) for both referrer and referee.
- **Fluent Integration**: Simplified API and "HasReferrals" trait for easy model integration.
- **Secure Tracking**: Reliable referral linking and conversion tracking.
- **Performance Analytics**: Real-time stats on referral volume, conversion rates, and earnings.
- **Abuse Prevention**: Built-in rate limiting and conditional reward checks.

## Installation

Refer is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Refer --packages
```

This will automatically:

- Run the migration for Refer tables.
- Register the `ReferServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/refer.php`

```php
return [
    // Referral code settings
    'code' => [
        'length' => 8,
        'prefix' => '',
        'uppercase' => true,
    ],

    // Reward configuration
    'rewards' => [
        'referrer' => [
            'type' => 'fixed',
            'amount' => 10.00, // $10.00
            'currency' => 'USD',
        ],
        'referee' => [
            'type' => 'fixed',
            'amount' => 5.00, // $5.00
            'currency' => 'USD',
        ],
    ],

    // Global limits
    'limits' => [
        'max_referrals' => 0, // 0 = unlimited
        'code_expiry_days' => 0,
    ],
];
```

## Basic Usage

### Static Facade

Use the `Refer` facade for quick access to core features:

```php
use Refer\Refer;

// Generate or retrieve a code for a user
$code = Refer::for($user->id);

// Track a conversion (e.g., on registration)
$referral = Refer::track($codeString, $newUserId);

// Reward manually (if not auto-rewarded)
Refer::reward($referral);

// Get global stats
$stats = Refer::getStats($user->id);
```

## Model Integration

Add the `HasReferrals` trait to your User model for a fluent API:

```php
use Refer\Traits\HasReferrals;

class User extends BaseModel
{
    use HasReferrals;
}
```

### Trait Usage

```php
// Get the user's referral code object
$code = $user->getReferralCode();

// Get the actual code string
echo $user->referral_code;

// Get the direct referral Link
echo $user->getReferralLink();

// Stats for this specific user
$stats = $user->getReferralStats();
// Returns: total_referrals, pending, rewarded, total_earnings (Money object)
```

## Use Cases

### SaaS Affiliate Program

Promote your SaaS by rewarding existing users with $20.00 credit for every friend who subscribes to a paid plan.

#### Implementation (Action Pattern)

In a production Anchor application, it is recommended to use an `Action` to handle user registration and referral tracking concurrently.

```php
namespace App\Actions\Auth;

use App\Models\User;
use Refer\Refer;
use App\Core\BaseAction;

class RegisterUserAction extends BaseAction
{
    /**
     * Handle user registration with optional referral tracking.
     */
    public function execute(array $data, ?string $referralCode = null): User
    {
        // 1. Create the user
        $user = User::create($data);

        // 2. Track referral if a code was provided (e.g., from a cookie or URL)
        if ($referralCode) {
            Refer::track($referralCode, $user->id);
        }

        return $user;
    }
}
```

### Subscription Reward

For SaaS, you might reward the referrer only after the referee's first successful payment:

```php
use Refer\Refer;

public function handleSuccessfulPayment(User $user)
{
    if ($user->wasReferred()) {
        $referral = $user->getReferralInstance();

        // Manual reward trigger
        if (!$referral->isRewarded()) {
             Refer::reward($referral);
        }
    }
}
```

### Sample Data (JSON)

The `referrals` table state:

```json
{
  "refid": "ref_abc123xyz",
  "referrer_id": 1,
  "referee_id": 42,
  "status": "rewarded",
  "referrer_reward": 20.0,
  "referee_reward": 0.0,
  "rewarded_at": "2026-01-02 11:05:00"
}
```

## Package Integrations

### Wallet Package (Financial Payouts)

`Refer` integrates with `Wallet` to automate reward payouts. When a reward is triggered, it performs a `credit()` operation.

```php
use Refer\Refer;
use Wallet\Wallet;

// End-to-End: Rewarding a user and verifying their new balance
public function completeReferral($referral)
{
    // 1. Trigger the reward (Refer package handles the Wallet credit)
    Refer::reward($referral);

    // 2. Integration: Verify the user's updated balance
    $newBalance = Wallet::forUser($referral->referrer_id)->balance();

    return "Referrer now has {$newBalance->format()}";
}
```

### Audit Package (Activity Trail)

Every referral event is logged automatically. You can also manually search these logs:

```php
use Audit\Audit;

// Finding all rewards processed today
$logs = Audit::history()
    ->where('event', 'referral.rewarded')
    ->where('created_at', '>=', now()->startOfDay())
    ->get();
```

## Advanced Features

### Retention & Conversion

By default, Refer can reward on signup. For SaaS, you might wait until their first purchase:

```php
// App/Config/refer.php
'conditions' => [
    'require_purchase' => true,
],
```

Then in your Purchase controller:

```php
use Refer\Refer;

if ($user->wasReferred()) {
    $referral = $user->getReferralInstance();
    Refer::reward($referral);
}
```

### Analytics & Reporting

The `ReferAnalytics` service provides deep insights into your growth funnel:

```php
$analytics = Refer::analytics();

// 1. Conversion Rate & Volume
$overview = $analytics->getOverview();
/**
 * Sample Data:
 * {
 *   "total_referrals": 1250,
 *   "successful_conversions": 850,
 *   "conversion_rate": 68.0,
 *   "total_payouts": 17000.00
 * }
 */

// 2. Growth Trends (last 30 days)
$trends = $analytics->getDailyTrends(30);
/**
 * Sample Data:
 * [
 *   {"date": "2026-01-01", "referrals": 45, "conversions": 30},
 *   {"date": "2026-01-02", "referrals": 52, "conversions": 38}
 * ]
 */

// 3. Top Promoters
$leaderboard = $analytics->getTopReferrers(5);
/**
 * Sample Data:
 * [
 *   {"user_id": 1, "name": "Alice", "referrals": 120, "earnings": 2400.00},
 *   {"user_id": 2, "name": "Bob", "referrals": 95, "earnings": 1900.00}
 * ]
 */
```

## Service API Reference

### Refer (Facade)

| Method              | Description                                           |
| :------------------ | :---------------------------------------------------- |
| `for($userId)`      | Quick generation/retrieval of a user's referral code. |
| `track($code, $id)` | Records a new referral conversion.                    |
| `reward($referral)` | Processes rewards for both parties.                   |
| `getStats($userId)` | Returns summarized performance for a user.            |
| `analytics()`       | Returns the `ReferAnalytics` service.                 |

### Analytics (Facade)

| Method              | Description                               |
| :------------------ | :---------------------------------------- |
| `getOverview()`     | Overall conversion and reward statistics. |
| `getDailyTrends($d)`| Track daily referral growth trends.       |
| `getTopReferrers($l)`| Leaderboard of most successful referrers. |

## Console Commands

| Command         | Description                             |
| :-------------- | :-------------------------------------- |
| `refer:stats`   | Visualize global referral health.       |
| `refer:cleanup` | Purge expired or unused referral codes. |

## Troubleshooting

| Error/Log                     | Cause                                      | Solution                                |
| :---------------------------- | :----------------------------------------- | :-------------------------------------- |
| "Referral code not valid"     | Code is expired, inactive, or maxed out.   | Check `is_active` and `max_uses` in DB. |
| "Self-referral not permitted" | User tried to use their own referral code. | Prevent this in the UI logic.           |
| "Referral already exists"     | User was already referred by someone else. | Each user can only have one referrer.   |

## Security Best Practices

- **Abuse Prevention**: Use the built-in `ReferRateLimiter` to prevent automated code generation or tracking spam.
- **Purchase Requirements**: For high-value rewards, always set `require_purchase` to `true` to prevent fake account signup loops.
- **Manual Audits**: Use `refer:stats` regularly to spot suspicious referral patterns (e.g., many referrals from a single IP).
- **Wallet Integrity**: Referral rewards are credited via the `Wallet` package, ensuring atomic transactions and audit logs.