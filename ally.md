# Ally

The Ally package provides a partner and reseller network management system that enables businesses to scale via tiered partnerships, a pre-paid distribution credit system, and partitioned client management.

## Features

- **Hierarchical Partner Profiles**: Detailed reseller profiles linked to system `User` accounts for authenticated access.
- **Tiered Partnership Logic**: Support for partner levels (e.g., Standard, Gold, Platinum) to govern custom pricing and permissions.
- **Distribution Credit Wallets**: Native integration with the **Wallet** package to manage pre-paid credits for service provisioning.
- **Portfolio Isolation**: Partitioned data views where resellers only manage clients they have onboarded.
- **Automated Provisioning**: Real-time service fulfillment (licenses, server resources) using credit balances to bypass manual payment gateways.
- **Event Hooks**: Hook into system events like `LowCreditEvent` to automate business logic when a partner's balance is low.
- **Low-Credit Notifications**: Automated alerts via the **Mail** package when a partner's balance drops below a critical threshold.

## Installation

Ally is a **package** that requires installation to initialize the partner network.

### Install the Package

```bash
php dock package:install Ally --packages
```

This will automatically:

- Run the migration for Ally tables.
- Register the `AllyServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/ally.php`

```php
return [
    'default_tier' => 'standard',
    'credit_conversion_rate' => 100, // e.g., 100 units = $1.00
    'auto_provision_wallet' => true,
    'low_credit_threshold' => 500, // Trigger notification at 500 units
    'urls' => [
        'dashboard' => 'partner/dashboard',
    ],
];
```

## Client Management

Ally works natively with the **Client** package to provide portfolio isolation.

### Relationship Structure

The connection is strictly maintained via the System `User` ID:

`Ally Reseller` -> `User (id: 1)` <- `Client (owner_id: 1)`

- **Ally** manages the partner's profile, wallet, and tier.
- **Client** records the end-customers belonging to that partner.

### Retrieving a Partner's Clients

Since clients are scoped by the `owner_id`, you can easily fetch all clients for a specific reseller.

```php
use Client\Client;

$user = $this->auth->user(); // The partner user

// 1. Get Reseller Profile
$reseller = Ally::findByUser($user->id);

// 2. Get Their Clients
$myClients = Client::getByReseller($user->id);

foreach ($myClients as $client) {
    echo $client->name; // "Acme Corp"
}
```

## Reseller Dashboard Integration

Ally enables you to build a dashboard for your partners to view their balance, tier, and performance.

### Retrieving Reseller Context

In your controller, retrieve the currently authenticated reseller.

```php
use Ally\Ally;

public function index()
{
    $user = $this->auth->user();

    $reseller = Ally::findByUser($user->id);

    // Get stats from Analytics service
    $stats = Ally::analytics()->resellerStats($reseller);

    // Get transaction history from Wallet
    $history = $reseller->wallet->transactions()->latest()->limit(10)->get();

    return $this->asView('partner.dashboard', compact('reseller', 'stats', 'history'));
}
```

### Displaying Data in View

You can then access all reseller properties directly.

```php
// Tier and Company
echo $reseller->company_name; // "Elite Global"
echo $reseller->tier; // "gold"

// Wallet Balance (from stats or direct relationship)
echo $stats['balance']; // 5000
// OR
echo $reseller->wallet->balance;

// Transaction History
foreach ($history as $txn) {
    echo "{$txn->type}: {$txn->amount} ({$txn->created_at})";
}
```

## Basic Usage

### Register a Reseller

The `Ally::make()` builder handles partner onboarding. You can use fluent methods for tiers.

```php
use Ally\Ally;

$reseller = Ally::make()
    ->user($userId)
    ->company('Elite Global Partners')
    ->platinum() // or ->gold(), ->standard()
    ->create();
```

### Funding and Provisioning

Methods accept both Reseller IDs and objects.

```php
// Add 50,000 credit units (Admin Action)
Ally::addCredits($reseller, 50000);

// Deduct credits for a service (Reseller Action)
if (Ally::provision($reseller, 200, 'Issue Pro License')) {
    // Deducted 200 credits successfully.
}
```

## Use Case Walkthrough

Ally is often the core of a "Channel Sales" strategy where your application is sold by third-party partners.

### Tier-Based Pricing & Profit Margins

In this scenario, we use the `calculateTierCost()` helper to determine the discounted rate.

```php
$reseller = Ally::findByUser($authId);
$basePrice = 1000; // Standard cost is 1,000 credits

// Calculate cost based on tier (Platinum: 30% off, Gold: 15% off)
$cost = $reseller->calculateTierCost($basePrice);

if (Ally::provision($reseller, $cost)) {
    // Reseller paid the discounted rate.
    Forge::make()->client($clientId)->create();
}
```

### Bulk Client Provisioning

Resellers often onboard multiple clients at once. Ally manages the ledger and ensures the partner has sufficient funds for the entire operation.

```php
$clients = $request->post('clients'); // Array of client data
$licenseCost = 500;
$totalCost = count($clients) * $licenseCost;

// 1. Verify total affordability first
if ($reseller->wallet->balance < $totalCost) {
    throw new Exception("Insufficient credits for bulk operation.");
}

// 2. Process batch
foreach ($clients as $data) {
    Ally::provision($reseller->id, $licenseCost);

    $client = Client::make()
        ->name($data['name'])
        ->reseller($reseller->user_id)
        ->create();

    Forge::make()->client($client->id)->create();
}
```

### Low-Credit Health Monitoring & Automation

The Ally package automatically monitors credit health. When `Ally::provision()` is called and the balance falls below the threshold defined in `App/Config/ally.php`, a `LowCreditEvent` is dispatched.

The package includes a built-in `SendLowCreditNotification` listener that handles email alerts. You can hook onto this event for additional logic.

**Hook onto the Event**

You can listen to this event to automate actions, such as suspending the reseller's ability to provision or notifying an account manager.

```php
// In a ServiceProvider or Event Listener
use Core\Event;
use Helpers\Log;
use Ally\Events\LowCreditEvent;

Event::listen(LowCreditEvent::class, function (LowCreditEvent $event) {
    // 1. Log the event for admin review
    Log::channel('ally')->info("Low credit warning for {$event->reseller->company_name}: {$event->balance}");

    // 2. Custom Automation: Pause high-value provisioning
    $event->reseller->updateMetadata('hold_provisioning', true);
});
```

- **Reseller Experience**: Receives an email (via `LowCreditNotification`) alerting them to top up.
- **Automation**: Your custom listener can preemptively pause services until the balance is restored.

### Monthly Partner Performance Report

You can use the Analytics Service to generate end-of-month reports for your admin dashboard. This helps identifying high-performing partners who might be eligible for a tier upgrade.

```php
use Ally\Ally;
use Ally\Enums\ResellerTier;
use System\Helpers\DateTimeHelper;

public function monthlyReport()
{
    // 1. Get top performers (explicit string formatting)
    $leaders = Ally::analytics()->topResellers(
        limit: 5,
        start: DateTimeHelper::now()->subMonth()->toDateTimeString(),
        end: DateTimeHelper::now()->toDateTimeString()
    );

    // 2. Get network health (total outstanding credits)
    $liability = Ally::analytics()->totalCreditsDistributed();

    // 3. Generate summary
    foreach ($leaders as $partner) {
        if ($partner['acquisitions'] > 100 && $partner['tier'] !== ResellerTier::PLATINUM->value) {
            // Suggest upgrade
            Log::channel('ally')->info("Partner eligible for upgrade: {$partner['company']}");
        }
    }

    return $this->asView('admin.reports.partners', compact('leaders', 'liability'));
}
```

## Advanced Features

### Reseller Tiers

Tiers govern your reseller logic (pricing, discounts, access). The package provides a `ResellerTier` enum for standardisation:

| Tier       | Value      | Description                                |
| :--------- | :--------- | :----------------------------------------- |
| `PLATINUM` | `platinum` | Highest volume partners, maximum discount. |
| `GOLD`     | `gold`     | Mid-tier partners with moderate benefits.  |
| `STANDARD` | `standard` | Default tier for new resellers.            |

You can use fluent helper methods for type-safe comparisons:

```php
if ($reseller->isPlatinum()) {
    // Apply VIP treatment
}
```

## Service API Reference

### Ally (Facade)

| Method                        | Description                                                              |
| :---------------------------- | :----------------------------------------------------------------------- |
| `make()`                      | Returns a fluent `AllyBuilder` for registration.                         |
| `findByUser($id)`             | Locates a reseller profile by associated User ID.                        |
| `findByRefid($refid)`         | Locates a reseller profile by its public reference.                      |
| `addCredits($reseller, $amt)` | Increases the credit balance of a reseller. Accepts object or ID.        |
| `provision($reseller, $cost)` | Deducts credits and checks for low balance alerts. Accepts object or ID. |
| `analytics()`                 | Returns the `AnalyticsManager` service.                                  |

### AnalyticsManager

| Method                               | Description                                                            |
| :----------------------------------- | :--------------------------------------------------------------------- |
| `topResellers($limit, $start, $end)` | Returns array of top partners.                                         |
| `tierSnapshot($start, $end)`         | Returns count of partners per tier. Supports `forReseller()` scoping.  |
| `creditTrends($start, $end)`         | Returns historical volume data. Supports fluent intervals and scoping. |
| `totalCreditsDistributed()`          | Returns total sum of all wallet balances.                              |
| `resellerStats($reseller)`           | Returns balance and tier stats for a specific partner.                 |
| `daily()`, `monthly()`, `yearly()`   | Chainable methods to set trend aggregation interval.                   |
| `forReseller($id)`                   | Chainable method to scope analytics to a specific partner.             |

### Reseller (Model)

| Method                     | Type        | Description                                  |
| :------------------------- | :---------- | :------------------------------------------- |
| `user()`                   | `relation`  | BelongsTo relationship to the User model.    |
| `scopeActive()`            | `scope`     | Filters for resellers with 'active' status.  |
| `scopePlatinum()`          | `scope`     | Filters for resellers in the Platinum tier.  |
| `scopeGold()`              | `scope`     | Filters for resellers in the Gold tier.      |
| `scopeStandard()`          | `scope`     | Filters for resellers in the Standard tier.  |
| `isPlatinum()`             | `bool`      | Checks if reseller is Platinum tier.         |
| `isGold()`                 | `bool`      | Checks if reseller is Gold tier.             |
| `isStandard()`             | `bool`      | Checks if reseller is Standard tier.         |
| `calculateTierCost($cost)` | `int`       | Returns cost after applying tier discount.   |
| `updateMetadata($k, $v)`   | `bool`      | Fluently updates a key in the metadata JSON. |
| `metadata`                 | `attribute` | JSON data for whitelabeling (logos, etc).    |

## Troubleshooting

| Error/Log               | Cause                                    | Solution                                  |
| :---------------------- | :--------------------------------------- | :---------------------------------------- |
| `InsufficientCredits`   | The cost exceeds the wallet balance.     | Add credits or reduce the provision cost. |
| "Notification not sent" | Mail driver is not configured properly.  | Check `App/Config/mail.php` settings.     |
| "Wrong owner_id"        | Client is linked to a different partner. | Verify attribution in the Client package. |

## Analytics

Ally provides a built-in `AnalyticsManager` with a fluent, chainable API to track reseller performance and distribution metrics.

### Fluent Time-Series Trends

You can track historical credit distribution trends using `daily()`, `monthly()`, or `yearly()` intervals.

```php
// Monthly credit distribution volume for the last 6 months
$trends = Ally::analytics()
    ->monthly()
    ->creditTrends('2023-01-01', '2023-06-30');

// Returns ['2023-01' => 75000, '2023-02' => 82000, ...]
```

### Scoped Analytics

Any analytical call can be scoped to a specific reseller. This is ideal for reseller-specific reports.

```php
// Get a specific reseller's credit volume trends
$partnerTrends = Ally::analytics()
    ->forReseller($resellerId)
    ->daily()
    ->creditTrends($start, $end);
```

### Snapshot Metrics

#### Tier Distribution Snapshot

Visualize the composition of your partner network.

```php
$stats = Ally::analytics()->tierSnapshot();

/* Sample Output:
[
    'platinum' => 12,
    'gold' => 45,
    'standard' => 120
]
*/
```

#### Global Credit Distribution

Track the total volume of credits currently held in partner wallets.

```php
$outstandingCredits = Ally::analytics()->totalCreditsDistributed();
// Returns (int): 2500000
```

## Automation

The Ally package includes automated credit monitoring via the `AllySchedule` class. It runs the `ally:check-credits` command daily to notify partners near their threshold.

## Security & Best Practices

- **Strict Wallet Modification**: The `addCredits()` method should never be exposed to public-facing reseller routes. It is exclusively for administrative or payment-webhook use.
- **Whitelabeling**: Use the `metadata` field on the `Reseller` model to store branding information. Your UI can then check if a user belongs to a reseller and dynamically skin the dashboard.
- **Tier Restrictions**: Beyond pricing, use tiers to control which **Forge** products a partner can distribute (e.g., Silver partners can only sell Basic licenses).

## Frequently Asked Questions

### How is Ally different from the Refer package?

**Ally** is designed for **B2B Resellers** and Partners who function as distributors. They have:

- A wallet balance for pre-purchasing inventory.
- A dashboard to manage their own clients.
- Tiered pricing privileges.

**Refer**, on the other hand, is a lightweight **Affiliate/Referral** system. It is for:

- Tracking invite links (e.g., `?ref=ben`).
- Rewarding users for signups (e.g., "Give $5, Get $5").
- Simple commission usage without client management.

**Can they works together?**
Yes. You can use **Refer** to acquire new partners (e.g., refer a friend to become a reseller), and then use **Ally** to manage that partner's lifecycle. They do not share code dependencies.

### Integration

#### Reseller Acquisition

In this workflow, an existing partner refers a new user who then becomes a sub-reseller.

**Generate Invite Link**
Existing partners share a `Refer` tracked link.

```php
$link = Refer::link(auth()->user(), 'become-partner');
// https://app.com/register?ref=BEN123
```

**Handle Registration**
When the new user registers, capture the referral code.

```php
public function register() {
    // 1. Create System User
    // Note: Use only() or explicit post() retrieval
    $user = User::create($this->request->only(['name', 'email', 'password']));

    // 2. Track Conversion (Refer Package)
    // This logs that BEN123 referred this new user
    Refer::complete($this->request->post('ref'), $user);

    // 3. Onboard as Reseller (Ally Package)
    $reseller = Ally::make()
        ->user($user)
        ->company($this->request->post('company_name'))
        ->standard()
        ->create();

    return $this->asView('dashboard', compact('reseller'));
}
```

**Reward the Referrer (Optional)**
You can use the `ReferralCompleted` event to give the referrer 1000 free credits.

```php
Event::listen(ReferralCompleted::class, function ($event) {
    if ($event->tag === 'become-partner') {
        // Find the referrer's reseller profile
        $referrer = Ally::findByUser($event->referrer->id);

        // Give bonus credits
        Ally::addCredits($referrer, 1000);
    }
});
```
