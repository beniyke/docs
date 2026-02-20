# Wave

**Wave** is a package designed to streamline subscription billing for your application. It handles the entire lifecycle of a subscription, from plan creation and trial management to recurring invoicing, tax calculation, and payment collection via multiple gateways.

## Features

- **Recurring Subscription Management**: Support for multiple plans, intervals (daily, weekly, monthly, yearly), trial periods, and quantity scaling.
- **Hybrid Payment System**: Seamlessly integrates with **Wallet** for balance-based billing and **Pay** for external payment collection (cards, bank transfers, etc.).
- **Smart Invoicing**: Automated invoice generation with PDF support, email dispatch, and tax calculation.
- **Coupon & Discount Engine**: Apply fixed or percentage-based discounts to subscriptions with usage limits and expiration handling.
- **Advanced Lifecycle Handling**: Built-in support for grace periods, subscription swaps, cancellations, and renewals.
- **Global Tax Support**: Automated tax logic based on user location or global configuration.

## Installation

Wave is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Wave --packages
```

This command will:

- Publish the `wave.php` configuration file.
- Run the migration for Wave tables.
- Register the `WaveServiceProvider`.
- Enable global helper function (`wave()`)

### Configure Settings

Edit `App/Config/wave.php` to match your business logic:

```php
return [
    'currency' => 'USD', // Default currency for plans and invoices

    /*
     * Payment Strategy
     * 'direct': Payments go directly to the invoice (Standard SaaS).
     * 'wallet': Payments fund a digital wallet, then the wallet pays the invoice (Marketplaces/Credits).
     */
    'payment_strategy' => 'direct',

    'trial_days' => 14,  // Default trial period for new subscriptions
    'grace_period_days' => 5, // Days to keep subscription active after payment failure

    /*
     * Tax Configuration
     * 'enabled': Toggle global tax calculation
     * 'rate': Default flat tax rate (if no region-specific rate found)
     * 'inclusive': Whether prices include tax (VAT style) or exclude it (US Sales Tax style)
     */
    'tax' => [
        'enabled' => true,
        'rate' => 15, // 15% VAT
        'inclusive' => false,
        'default_country' => 'US', // Fallback for tax calculation
    ],

    // Dynamic URLs for notifications
    'invoice' => [
        'prefix' => 'INV-',
        'url' => '/billing/invoices', // {refid} appended automatically
        'subscription_url' => '/billing/subscription',
        'payment_method_url' => '/billing/payment-method',
        'footer' => 'Thank you for your business!',
    ],

    'affiliate' => [
        'commission_rate' => 10, // 10% commission on referrals
    ],
];
```

## Architecture & Lifecycle

To understand how Wave orchestrates your business, view it as an ecosystem where each component has a specific role.

### The Blueprint Layer

#### Models & Config

Everything starts with **Plans** (Recurring) and **Products** (One-Time). These define the base price and interval. **TaxRates** are configured here to define the geography of your billing rules.

### The Relationship Layer

#### Subscription & Affiliate

When a user subscribes, the `SubscriptionBuilder` creates the "Contract". If they arrived via a referral link, the `AffiliateManager` tags them. No money moves yet; it is just a record.

### The Workhorse Layer

#### Invoices & Taxes

The `InvoiceManager` handles "Billing events." It looks at the Agreement (Subscription) and creates a snapshot of the debt. It hands this to the `TaxManager`, which appends regional taxes based on the user's location.

### The Money Layer

#### Wallet vs. Pay

When collecting payment, Wave uses a hierarchical strategy:

- **Wallet**: Checks internal balance first. If sufficient, the invoice is paid instantly.
- **Pay Service**: If the wallet is empty, Wave initializes an external checkout (Stripe, PayPal, etc.).

### The Conversion Layer

#### Feedback Loop

Once payment is confirmed, the `AffiliateManager` calculates commission for the partner, and the `AnalyticsManager` updates key performance metrics:

- **MRR (Monthly Recurring Revenue)**: The total predictable revenue your SaaS generates each month. Wave automatically "normalizes" yearly or weekly plans into a monthly value for this calculation.
- **Churn Rate**: The percentage of subscribers who canceled in the last 30 days. It measures customer retention and is the most critical indicator of "product-market fit."
- **LTV (Lifetime Value)**: The total revenue collected from a single customer over their entire relationship with your app. It helps you determine how much you can afford to spend on customer acquisition.

## Core Concepts

Wave distinguishes between two types of billing:

- **Subscriptions (Recurring)**: Plans that auto-renew (e.g., "Monthly Pro"). Managed via `PlanManager` and `SubscriptionManager`.
- **Products (One-Time)**: Individual items or services charged once (e.g., "Setup Fee", "Consultation"). Managed via direct invoicing or `Product` models (if using inventory).

## Basic Usage

### Recurring Subscriptions

#### Creating a Plan

Plans define what your users are subscribing to.

```php
use Wave\Wave;

$plan = Wave::plan()->make()
    ->name('Pro Plan')
    ->slug('pro-monthly')
    ->price(29.00) // Amount in dollars ($29.00)
    ->currency('USD')
    ->monthly() // or ->interval('month')
    ->description('Perfect for professionals.')
    ->save();
```

### Global Helper

The Wave package provides a convenient global helper for accessing the manager service.

```php
// Get WaveManagerService instance
$wave = wave();

// Alias for accessing sub-services
$plans = wave()->plan();
$subscriptions = wave()->subscriptions();
```

### Subscribing a User

To subscribe a user (or any model) to a plan:

```php
use Wave\Wave;

$subscription = Wave::newSubscription()
    ->for($user) // Automatically sets owner_id and owner_type
    ->plan('pro-monthly')
    ->trialDays(14)
    ->quantity(1)
    ->start();
```

### Checking Status

Check if a user has an active subscription:

```php
if (Wave::subscriptions()->hasActiveSubscription($user->id, 'user')) {
    // Grant access
}

```

### One-Time Products

You can define reusable products for one-time purchases, though explicit `Product` creation is optional if you just need ad-hoc invoicing.

```php
Wave::product()->make()
    ->name('Setup Fee')
    ->description('One-time server configuration')
    ->price(500.00) // $500.00
    ->currency('USD')
    ->create();
```

### One-Time Payments

For non-recurring charges, you generate an invoice directly. You can link it to a conceptual "Product" or just charge an ad-hoc amount.

```php
use Wave\Wave;
use Helpers\DateTimeHelper;

// Create an invoice for a one-time service
$invoice = Wave::invoices()->make()
    ->for($user)
    ->amount(50.00) // $50.00
    ->currency('USD')
    ->description('Lifetime Access Fee')
    ->discount('BLACKFRIDAY25')
    ->dueInDays(7) // Optional: Set due date to 7 days from now
    // ->dueNow() // Or due immediately
    ->create();

// Attempt payment immediately
$paid = Wave::invoices()->attemptPayment($invoice);

if ($paid) {
    // Check if a redirect is required
    $invoice->refresh(); // Refresh to get latest metadata
    $checkoutUrl = $invoice->metadata['checkout_url'] ?? null;

    if ($checkoutUrl) {
         // Redirect user to payment gateway (Stripe/PayPal/etc)
         return redirect($checkoutUrl, [], false);
    }
```

### Manual Payment Overrides

In some cases (e.g., bank transfers or check payments), you may need to mark an invoice as paid manually.

```php
$invoice = Wave::invoices()->find('INV-2024-0042');

// Mark as paid with optional driver and transaction ID
Wave::invoices()->markAsPaid($invoice, driver: 'manual_wire', transactionId: 'TXN_998877');
```

## Automation& CLI

Wave includes built-in commands for the **Dock** CLI to handle recurring tasks like renewal reminders and processing overdue payments.

### High-Level Commands

| Command       | Description                                        |
| :------------ | :------------------------------------------------- |
| `wave:remind` | Send email reminders for upcoming renewals.        |
| `wave:renew`  | Create invoices and process payments for renewals. |

### Renewal Reminders

Notify users whose subscriptions are about to renew.

```bash
# Send reminders for subscriptions renewing in 3 days (default)
php dock wave:remind

# Send reminders for renewals in 7 days
php dock wave:remind --days=7
```

### Renewal Processing

Automatically process renewals that have reached their period end. This command finds active subscriptions where the `current_period_end` has passed, generates a new invoice, and attempts payment.

```bash
# Process all pending renewals
php dock wave:renew
```

**Production Tip**: Set up a CRON job to run these commands daily.

```cron
0 0 * * * php /path/to/project/dock wave:renew
0 0 * * * php /path/to/project/dock wave:remind
```

### Cancellation Logic

You can control exactly when a subscription should end.

```php
// 1. Cancel at period end (Default - Graceful)
// Set status to active but sets ends_at = current_period_end
Wave::subscriptions()->cancel($subId, atPeriodEnd: true);

// 2. Cancel immediately (Nuclear)
// Sets status to 'canceled' and ends_at = now
Wave::subscriptions()->cancel($subId, atPeriodEnd: false);
```

## Advanced Usage

### Swapping Plans

Upgrade or downgrade a user's subscription instantly. Wave handles the proration automatically and updates the billing cycle.

```php
// Swap to Enterprise plan
Wave::subscriptions()->swap($subscription->id, 'enterprise-yearly');
```

### Managing Quantity

Useful for "per-seat" billing models.

```php
// Add 2 more seats
Wave::subscriptions()->updateQuantity($subscription->id, 5);
```

### Creating Coupons

Before applying coupons, you must create them. Wave supports percentage-based and fixed-amount coupons.

```php
use Wave\Models\Coupon;
use Wave\Enums\CouponType;
use Wave\Enums\CouponDuration;

// Create a 20% off forever coupon
Wave::coupons()->make()
    ->code('FOREVER20')
    ->name('20% Off Forever')
    ->percent(20) // 20% off (applies to any currency)
    ->forever()
    ->create();

// Create a $10 off once coupon
Wave::coupons()->make()
    ->code('WELCOME10')
    ->name('$10 Welcome Bonus')
    ->fixed(10.00, 'USD') // $10.00 in USD (default is USD if omitted)
    ->once()
    ->maxRedemptions(100)
    ->create();
```

### Applying Coupons

Apply a discount code to an active subscription.

```php
use Wave\Models\Coupon;
use Wave\Exceptions\CouponExpiredException;

try {
    Wave::coupons()->applyToSubscription($subscription, 'BLACKFRIDAY25');
} catch (CouponExpiredException $e) {
    // Handle invalid coupon
}
```

### Manual Invoicing

While Wave handles recurring invoices automatically, you can trigger one manually:

```php
$invoice = Wave::invoices()->createFromSubscription(
    $subscription,
    description: 'Ad-hoc charge for extra resources'
);

// Attempt to collect payment immediately
$paid = Wave::invoices()->attemptPayment($invoice);

if ($paid) {
    // Payment successful via Wallet or Pay
}
```

## Service API Reference

### Wave Facade

The `Wave\Wave` facade provides a unified entry point for all services.

| Method               | Description                              |
| :------------------- | :--------------------------------------- |
| `subscribe(...)`     | Quick alias to start a new subscription. |
| `newSubscription()`  | Returns a fluent `SubscriptionBuilder`.  |
| `plan()`             | Returns the `PlanManager`.               |
| `product()`          | Returns the `ProductManager`.            |
| `subscriptions()`    | Returns the `SubscriptionManager`.       |
| `invoices()`         | Returns the `InvoiceManager`.            |
| `coupons()`          | Returns the `CouponManager`.             |
| `taxes()`            | Returns the `TaxManager`.                |
| `analytics()`        | Returns the `AnalyticsManager`.          |
| `affiliates()`       | Returns the `AffiliateManager`.          |
| `findPlan(id)`       | Helper to find plan by ID/Slug.          |
| `findProduct(id)`    | Helper to find product by ID/RefID.      |
| `findSubscription()` | Helper to find subscription by ID/RefID. |

### SubscriptionManager

Access via `Wave::subscriptions()`.

| Method                       | Description                                  |
| :--------------------------- | :------------------------------------------- |
| `find(id)`                   | Find a subscription by ID.                   |
| `swap(id, planId)`           | Change the plan for a subscription.          |
| `cancel(id, atPeriodEnd)`    | Cancel subscription (immediate or deferred). |
| `updateQuantity(id, qty)`    | Update the quantity of subscribed items.     |
| `hasActiveSubscription(...)` | Check if a user is subscribed.               |
| `calculatePeriodEnd(...)`    | Helper to determine next billing date.       |
| `sendRenewalReminders(days)` | Send email reminders for expiring subs.      |

### InvoiceManager

Access via `Wave::invoices()`.

| Method                        | Description                                                |
| :---------------------------- | :--------------------------------------------------------- |
| `createFromSubscription(...)` | Generate a new invoice from subscription data.             |
| `find(id)`                    | Retrieve an invoice by its ID.                             |
| `attemptPayment(invoice)`     | **Critical**: Attempts payment via Wallet first, then Pay. |
| `markAsPaid(invoice...)`      | Manually mark an invoice as paid with driver details.      |
| `calculateTax(invoice)`       | Applies configured tax rules to the invoice.               |

### TaxManager

Access via `Wave::taxes()`.

| Method     | Description                      |
| :--------- | :------------------------------- |
| `make()`   | Start a fluent tax rate builder. |
| `find(id)` | Find a tax rate by ID or RefID.  |

### CouponManager

Access via `Wave::coupons()`.

| Method                     | Description                                          |
| :------------------------- | :--------------------------------------------------- |
| `findByCode(code)`         | Retrieve a coupon by its unique code.                |
| `applyToSubscription(...)` | Link a coupon to a subscription for future invoices. |

### AffiliateManager

Access via `resolve(AffiliateManager::class)` (or helper if available).

| Method                | Description                                      |
| :-------------------- | :----------------------------------------------- |
| `recordReferral(...)` | Link a new user to an affiliate code.            |
| `onConversion(...)`   | Calculate commission after a successful payment. |
| `getReferrals(code)`  | Get all referrals for a specific affiliate code. |

#### Implementing Affiliates

To implement an affiliate program:

- **Capture the Referral**: When a user registers with an `?ref=CODE` query parameter.

- **Record It**: Call `recordReferral` during registration.
- **Process Commission**: The system automatically handles `onConversion` when `ProcessPaymentSuccess` fires (ensure your listener is set up if customizing).

```php
// In your RegisterController
use Wave\Services\AffiliateManager;

public function register(Request $request, AffiliateManager $affiliates)
{
    // ... create user ...

    if ($code = $request->get('ref')) {
        try {
            $affiliates->recordReferral($code, $user->id, 'user');
        } catch (\Exception $e) {
            // Invalid code, ignore or log
        }
    }
}
```

#### Listing Referrals

You can retrieve all referrals associated with a specific affiliate code for reporting.

```php
use Wave\Wave;

// Get referrals via the Wave facade
$referrals = Wave::affiliates()->getReferrals('PARTNER2024');

foreach ($referrals as $referral) {
    echo "Referred User: {$referral->referred_owner_id} - Status: {$referral->status}";
}
```

## Configuration & Strategies

### Payment Strategy

Wave allows you to choose how payments are routed via the `payment_strategy` config.

- **Direct (`'direct'`)**: The standard approach. When a user pays an invoice, the money is marked against that invoice immediately. Best for simple SaaS apps.
- **Wallet (`'wallet'`)**: The "Centralized Ledger" approach. When a user pays, the money funds their internal `Wallet` balance first. Then, the system attempts to pay the invoice using that balance. Best for platforms where users maintain a balance or credits.

To verify the strategy in code:

```php
$strategy = config('wave.payment_strategy', 'direct');
```

### Tax Logic

Wave's tax system is hierarchical:

- **Region Specific**: The system looks for a `Wave\Models\TaxRate` matching the user's `billing_country` (and optional `billing_state`).
- **Global Fallback**: If no specific rate is found, it uses `wave.tax.rate` from the config.

To populate tax rates:

```php
use Wave\Models\TaxRate;

Wave::taxes()->make()
    ->country('DE')
    ->rate(19.0) // 19% VAT
    ->name('VAT (Germany)')
    ->exclusive() // or ->inclusive()
    ->create();
```

### Understanding Tax Types

When creating tax rates, you can specify whether the tax is **Exclusive** (default) or **Inclusive**.

#### Exclusive Tax (Price + Tax)

Tax is added on top of the product price. This is common in the **US** and for **B2B** transactions.

- **Example**: Product is $100. Tax is 10%.
- **Calculation**: $100 + ($100 \* 0.10) = $110 Total.
- **Usage**: Use `->exclusive()`.

#### Inclusive Tax (Price includes Tax)

Tax is already included in the display price. The system extracts the tax amount from the total. This is standard in the **EU/UK** (VAT) and for **B2C** transactions where the customer expects to pay exactly the displayed price.

- **Example**: Product is $110. Tax is 10% (inclusive).
- **Calculation**: Tax = $110 - ($110 / 1.10) = $10. Base Price = $100.
- **Usage**: Use `->inclusive()`.

### Trials & Grace Periods

- **Trial Days**: Defined globally in config (`trial_days`) or overridden per-subscription using the fluent builder.
  - _Usage_: `->trialDays(14)` sets the trial period to 14 days, overriding the plan default.
- **Grace Period**: If a renewal payment fails, the subscription enters `past_due`. The user retains access for a duration defined in the global config `grace_period_days`. This is not set per-subscription but is a system-wide policy for failed payments.

```php
// Subscription with explicit 30-day trial
$subscription = Wave::newSubscription()
    ->for($user)
    ->plan($plan->id)
    ->trialDays(30) // Overrides plan's default trial
    ->start();
```

## Integrations & Payment Flow

Wave's payment logic is robust and fail-safe. When `attemptPayment()` is called:

- **Wallet Check**: It checks the user's internal `Wallet` balance.

  - **Success**: Balance is deducted, invoice marked PAID.
  - **Failure**: Proceed to step 2.

- **Pay Fallback**: It initializes a payment via the `Pay` package.
  - This generates a secure payment link (e.g., Stripe Checkout, Paystack Page).
  - User pays via the link -> Webhook confirms payment -> Invoice marked PAID.

### Notifications

Wave automatically sends email notifications for key events using the `Mail` package.

- `InvoiceGeneratedNotification`: Sent when a new invoice is created. Contains a direct link to pay/view.
- `PaymentSuccessNotification`: Receipt sent after successful payment.
- `PaymentFailedNotification`: Alert sent when wallet and fallback methods both fail.
- `RenewalReminderNotification`: Sent `wave.reminder_days` before a subscription renews.

## Fetching & Querying

Since `Subscription` and `Invoice` are standard Eloquent models, you can query them directly for reporting.

### Subscriptions

```php
use Wave\Models\Subscription;

// Fetch all active subscriptions
$active = Subscription::active()->get();

// Get subscriptions on trial
$trialing = Subscription::trialing()->get();

// Get subscriptions that have been canceled
$canceled = Subscription::canceled()->get();

// Get subscriptions with failed payments
$pastDue = Subscription::pastDue()->get();

// Find subscriptions renewable in the next 3 days
$renewingSoon = Subscription::isRenewableInDays(3)->get();
```

### Subscription Helpers

The `Subscription` model includes convenient helper methods to check status and remaining time.

```php
// Check if subscription grants access (Active, Trialing, or Canceled but in grace period)
if ($subscription->isCurrentlyActive()) {
    // User has access
}

// Get remaining days of access (Current cycle or grace period)
// Returns 0 if expired
$daysLeft = $subscription->daysLeft();
echo "You have {$daysLeft} days remaining.";

// Get a human-readable status label
// Returns: 'Active', 'Trialing', 'Past Due', 'Canceled', or 'Canceled (Grace Period)'
echo $subscription->displayStatus();
```

### Invoices

```php
use Wave\Models\Invoice;

// Get all open (unpaid) invoices
$unpaid = Invoice::open()->get();

// Get paid invoices
$paid = Invoice::paid()->get();

// Get overdue invoices
$overdue = Invoice::overdue()->get();
```

### Generic Model Scopes

Most Wave models (`Plan`, `Product`, `Coupon`, `Subscription`) support fluent status checks.

```php
// Plans
// 1. Subscription Scopes
// -----------------------
// Get all active subscriptions
$active = Subscription::isActive()->get();

// Get subscriptions on trial
$trialing = Subscription::isOnTrial()->get();

// Get subscriptions belonging to a specific owner
$userSubs = Subscription::owner($user->id)->get();

// Get subscriptions renewable in the next X days
$renewingSoon = Subscription::isRenewableInDays(3)->get();

// 2. Plan Scopes
// --------------
// Get all active plans
$activePlans = Plan::isActive()->get();

// Get inactive plans
$inactivePlans = Plan::isInactive()->get();

// Get monthly/yearly plans
$daily = Plan::isDaily()->get();
$weekly = Plan::isWeekly()->get();
$monthly = Plan::isMonthly()->get(); // 1 month
$quarterly = Plan::isQuarterly()->get(); // 3 months
$biannual = Plan::isBiannual()->get(); // 6 months
$yearly = Plan::isYearly()->get(); // 1 year

// 3. Product Scopes
// -----------------
// Get all active products
$activeProducts = Product::isActive()->get();

// Get inactive products
$inactiveProducts = Product::isInactive()->get();

// 4. Coupon Scopes
// ----------------
// Get all active coupons
$activeCoupons = Coupon::isActive()->get();

// Get all expired coupons
$expiredCoupons = Coupon::isExpired()->get();
```

## Analytics Service

The `Wave::analytics()` service provides a comprehensive suite of methods for retrieving SaaS metrics and reporting data. This service is essential for building admin dashboards, user reports, and monitoring business health.

### Monthly Recurring Revenue (MRR)

Calculates the total monthly normalized revenue from all active subscriptions.

- **Use Case**: Displaying the "Headline" MRR figure on an admin dashboard.
- **Method**: `mrr()`
- **Returns**: `Money\Money` (Immutable Money Object)

```php
$mrr = Wave::analytics()->mrr();
// Returns Money object
echo $mrr->formatSimple(); // Outputs: $5,000.00
echo $mrr->getAmount(); // Outputs: 500000
```

### Total Revenue (Date Range)

Calculates the total revenue collected from paid invoices within an optional date range.

- **Use Case**: Generating financial reports for a specific month or year.
- **Method**: `revenue(?string $start, ?string $end)`
- **Returns**: `Money\Money`

```php
// Total all-time revenue
$total = Wave::analytics()->revenue();
echo $total->formatSimple(); // $15,400.00

// Revenue for January 2024
$january = Wave::analytics()->revenue('2024-01-01', '2024-01-31');
```

### Active Subscribers

Counts the total number of users with an `active` or `trialing` subscription status.

- **Use Case**: Showing user growth metrics.
- **Method**: `activeSubscribers()`
- **Returns**: `int`

```php
// Count total active seats (Active + Trialing)
$count = Wave::analytics()->activeSubscribers();

// Count only subscribers currently on trial
$trialingCount = Wave::analytics()->trialingSubscribers();
```

### Churn Rate

Calculates the percentage of subscribers who canceled in the last X days (default 30) relative to the active count at the start of that period.

- **Use Case**: Monitoring retention health over different timeframes.
- **Method**: `churnRate(int $days = 30)`
- **Returns**: `float` (Percentage)

```php
// Monthly Churn (Default)
$monthlyChurn = Wave::analytics()->churnRate();

// 90-Day Churn
$quarterlyChurn = Wave::analytics()->churnRate(90);
```

### Historical Chart Data

Retrieves time-series data for charting libraries (like Chart.js or ApexCharts). Supports `revenue`, `new_subscriptions`, and `cancellations`.

- **Use Case**: Populating line or bar charts for trends over time.
- **Method**: `getHistory(string $metric, string|array $range = '30d')`
- **Metrics**: `'revenue'`, `'new_subscriptions'`, `'cancellations'`
- **Ranges**: `'7d'`, `'30d'`, `'90d'`, `'year'`, or `['YYYY-MM-DD', 'YYYY-MM-DD']`
- **Returns**: `array`

#### Fluent API (Recommended)

```php
// Get revenue history
$revenue = Wave::analytics()->getHistory()->revenue('7d');

// Get new subscriptions history
$newSubs = Wave::analytics()->getHistory()->newSubscriptions('30d');

// Get cancellations history
$churn = Wave::analytics()->getHistory()->cancellations('year');
```

#### Standard API

```php
$data = Wave::analytics()->getHistory('revenue', '7d');

/* Returns:
[
'labels' => ['Jan 01', 'Jan 02', 'Jan 03', ...],
'values' => [15000, 20000, 0, ...] // values in cents or count
]
*/
```

### Subscriber Statistics (LTV)

Retrieves Life-Time Value (LTV) and active subscription count for a specific owner (User/Team).

- **Use Case**: Showing a user's billing summary on their profile.
- **Method**: `subscriberStats(int|string $ownerId)`
- **Returns**: `array`

```php
$stats = Wave::analytics()->subscriberStats($user->id);

/* Returns:
[
    'ltv' => Money object, // Use $stats['ltv']->formatSimple() -> $1,250.00
    'active_subscriptions' => 1
]
*/
```

### Product Statistics

Retrieves the top-performing plans based on active subscription counts.

- **Use Case**: Identifying most popular plans.
- **Method**: `productStats()`
- **Returns**: `array`

```php
$stats = Wave::analytics()->productStats();

/* Returns:
[
    'top_plans' => [
        ['name' => 'Pro Monthly', 'count' => 150],
        ['name' => 'Basic Yearly', 'count' => 45],
        ...
    ]
]
*/
```

### Coupon Statistics

Retrieves usage metrics for coupons.

- **Use Case**: Analyzing marketing campaign performance.
- **Method**: `couponStats()`
- **Returns**: `array`

```php
$stats = Wave::analytics()->couponStats();

/* Returns:
[
    'total_redemptions' => 342,
    'active_coupons' => 5
]
*/
```

### Invoice Statistics

Retrieves a high-level overview of invoice statuses.

- **Use Case**: Admin overview of billing operations.
- **Method**: `invoiceStats()`
- **Returns**: `array`

```php
$stats = Wave::analytics()->invoiceStats();

/* Returns:
[
    'paid_count' => 1205,
    'unpaid_count' => 15,
    'overdue_count' => 3,
    'total_collected' => Money object // Use ->formatSimple() -> $15,400.00
]
*/
```

### Affiliate Statistics

Retrieves performance metrics for the affiliate program.

- **Use Case**: Monitoring partner contributions and commission payouts.
- **Method**: `affiliateStats(?string $affiliateCode = null)`
- **Returns**: `array`

```php
// Global Affiliate Stats
$globalStats = Wave::analytics()->affiliateStats();
/* Returns:
[
    'active_affiliates' => 12,
    'total_conversions' => 450,
    'total_commission' => Money object
]
*/

// Stats for a Specific Partner
$partnerStats = Wave::analytics()->affiliateStats('PARTNER2024');
/* Returns:
[
    'total_conversions' => 45,
    'total_commission' => Money object
]
*/
```

## Automation

The Wave package uses automated scheduling to handle subscription renewals and send reminders. These tasks are automatically registered in the framework scheduler:

```php
// packages/Wave/Schedules/SubscriptionRenewalSchedule.php
namespace Wave\Schedules;
 
use Cron\Interfaces\Schedulable;
use Cron\Schedule;
 
class SubscriptionRenewalSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('wave:renew')
            ->hourly();
 
        $schedule->task()
            ->signature('wave:renewal-reminders')
            ->daily();
    }
}
```

## Security & Best Practices

- **Idempotency**: Wave generates a unique `refid` for every invoice. This is used as the `Idempotency-Key` or `Reference` in all payment gateways to prevent double-charging.
- **Atomic Operations**: All plan swaps and renewals are wrapped in database transactions.
- **Public URLs**: Never expose internal IDs (integers). Use the `refid` (UUID-like string) for all public-facing routes (e.g., `/invoices/inv_x9s8d7f`).
