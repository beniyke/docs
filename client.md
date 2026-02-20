# Client

The Client package provides a robust customer lifecycle management system. It handles standalone clients, reseller-managed portfolios, and deep integration with licensing and support systems.

## Features

- **Multi-Tenant Lifecycle**: Manage standalone clients and those owned by resellers (Allies).
- **Binding & Identity**: Optional 1:1 binding between client entities and system `User` accounts.
- **Reference-Based Security**: Automatic generation of secure `refid`s for sensitive public exposure.
- **State Machine**: Fluent API for transitioning between Active, Pending, Suspended, and Inactive states.
- **Extensible Metadata**: JSON-based storage for industry-specific or custom client attributes.
- **Reseller Support**: Built-in logic for attribution and management via the reseller network.

## Installation

Client is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Client --packages
```

This will automatically:

- Run the migration for Client tables.
- Register the `ClientServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/client.php`

```php
use Client\Enums\ClientStatus;

return [
    'default_status' => ClientStatus::PENDING,
    'refid_prefix' => 'CL-',
    'auto_bind_user' => false,
];
```

## Basic Usage

### Register a New Client

Use the `Client` facade for a fluent registration experience:

```php
use Client\Client;

$client = Client::make()
    ->name('Acme Corporation')
    ->email('billing@acme.com')
    ->metadata(['segment' => 'Enterprise'])
    ->create();
```

### Reseller Attribution

Link a client to a managing reseller (Ally):

```php
$client = Client::make()
    ->name('Global Logistics')
    ->email('ops@global.log')
    ->reseller($allyId)
    ->create();
```

### Change Status

```php
Client::activate($clientId);
Client::suspend($clientId);
```

## Advanced Features

### Query Scopes

Efficiently filter your client database using built-in model scopes:

```php
use Client\Models\Client as ClientModel;

// Status Scopes
$active = ClientModel::active()->get();
$pending = ClientModel::pending()->get();
$suspended = ClientModel::suspended()->get();
$inactive = ClientModel::inactive()->get();

// Ownership Scopes
$standalone = ClientModel::standalone()->get(); // No reseller
$reselled = ClientModel::reselled()->get();     // Managed by reseller
```

### Client Analytics

Monitor acquisition trends and portfolio health using a fluent, chainable API:

```php
$analytics = Client::analytics();

// 1. Fluent Growth Trends
$growth = $analytics->monthly()->signupTrends('2023-01-01', '2023-06-30');

// 2. Scoped Analytics (View growth for a specific reseller)
$resellerGrowth = $analytics->forReseller($id)->daily()->signupTrends($start, $end);

// 3. Portfolio Health
$breakdown = $analytics->statusBreakdown();
// Returns: ['active' => 120, 'pending' => 45, 'suspended' => 5]

$segments = $analytics->segmentation();
/* Returns:
[
    'standalone' => ['count' => 50, 'percentage' => 33.33],
    'resold' => ['count' => 100, 'percentage' => 66.67]
]
*/
```

## Service API Reference

### Client (Facade)

| Method               | Description                                       |
| :------------------- | :------------------------------------------------ |
| `make()`             | Returns a fluent `ClientBuilder`.                 |
| `findByRefid($id)`   | Locates a client by secure reference string.      |
| `getByReseller($id)` | Retrieves all clients managed by a specific Ally. |
| `activate($id)`      | Transitions a client to the Active state.         |
| `analytics()`        | Returns the `AnalyticsManager` service.           |

### AnalyticsManager

| Method                             | Description                                                       |
| :--------------------------------- | :---------------------------------------------------------------- |
| `statusBreakdown($start, $end)`    | Returns client counts grouped by status.                          |
| `growth($start, $end)`             | Returns total new clients in period.                              |
| `segmentation($start, $end)`       | Returns standalone vs. resold client metrics.                     |
| `signupTrends($start, $end)`       | Returns time-series signup data. Supports fluent intervals.       |
| `daily()`, `monthly()`, `yearly()` | Chainable methods to set trend aggregation interval.              |
| `forReseller($id)`                 | Chainable method to scope analytics to a specific reseller/owner. |

### ClientBuilder (Fluent)

| Method            | Description                                  |
| :---------------- | :------------------------------------------- |
| `name($name)`     | Sets the corporate or individual name.       |
| `email($email)`   | Sets the primary billing/contact email.      |
| `reseller($id)`   | Attributes the client to a reseller.         |
| `metadata($data)` | Merges custom JSON data into the record.     |
| `create()`        | Persists the client and generates a `refid`. |

### Client (Model)

| Attribute  | Type      | Description                               |
| :--------- | :-------- | :---------------------------------------- |
| `refid`    | `string`  | Unique, non-sequential public identifier. |
| `status`   | `Enum`    | Active, Pending, Suspended, Inactive.     |
| `owner_id` | `integer` | ID of the managing Ally (if applicable).  |

## Package Integrations

The Client package is designed to be the central identity hub for other Anchor packages.

### Forge (Licensing)

Clients can own multiple licenses. To retrieve licenses for a client, use the `Forge` facade:

```php
use Forge\Forge;

$licences = Forge::licences()->getByClient($clientId);
```

### Wave (E-commerce & Billing)

Billing history and active subscriptions are managed via `Wave`:

```php
use Wave\Wave;

$invoices = Wave::invoices()->getByClient($clientId);
$subscriptions = Wave::subscriptions()->getByClient($clientId);
```

### Ally (Resellers)

If a client is managed by a reseller, the `owner_id` links it to an `Ally`. Resellers can provision services for their clients using their own credit balance.

```php
use Ally\Ally;
use Forge\Forge;

// 1. A reseller provisions a license for their client
$reseller = Ally::findByUser($authUserId);
$cost = 500; // 500 credit units

if (Ally::provision($reseller, $cost, 'Pro License for ' . $client->name)) {
    // 2. If credits are successfully deducted, mint the license contextually
    Forge::make()
        ->client($client)
        ->product($productId)
        ->create();
}
```

## Historical Data & Activity

While the Client package focuses on current state, integrations allow you to reconstruct historical activity:

- **Activation History**: Use `Forge::analytics()->forClient($id)->activationTrends()` for time-series data.
- **Billing History**: Use `Wave::analytics()->forClient($id)->revenueTrends()` for historical spend.
- **Audit Logs**: The framework's internal audit system tracks all status changes (e.g., from `Pending` to `Active`).

## FAQ

### Can a client use Forge without an Ally?

**Yes.** While many clients are onboarded via resellers (Allies), the Client package supports "Standalone" entities. You can create a client and mint licenses for them directly without any reseller attribution.

**The Workflow:**

- **Creation**: Omit the `reseller()` call.
- **Payment**: Use **Wave** or **Pay** directly for the invoice (bypassing the Ally credit system).
- **Fulfillment**: Mint the license via `Forge` once payment is confirmed.

```php
use Client\Client;
use Forge\Forge;
use Wave\Wave;

// 1. Create a standalone client
$client = Client::make()
    ->name('Independent Corp')
    ->email('info@independent.com')
    ->create();

// 2. Create invoice for a one-time product
$invoice = Wave::invoices()->make()
    ->for($client)
    ->amount(150)
    ->currency('USD')
    ->create();

// 3. Attempt payment (Wallet fallback or External Checkout)
if (Wave::invoices()->attemptPayment($invoice)) {
    // 4. Fulfillment: Activate client and mint/activate the license
    Client::activate($client->id);

    $licence = Forge::make()
        ->client($client)
        ->product($productId)
        ->create();

    Forge::activate($licence->id, $client);
}
```

## Troubleshooting

| Error/Log                  | Cause                                    | Solution                                      |
| :------------------------- | :--------------------------------------- | :-------------------------------------------- |
| "Duplicate Email"          | Email is already registered to a client. | Verify client directory or use `findByEmail`. |
| "Invalid Reseller"         | Provided `owner_id` does not exist.      | Ensure Ally is registered before assignment.  |
| "Status Transition Denied" | Illegal state change detected.           | Check `ClientStatus` transitions logic.       |

## Security Best Practices

- **Always Use Refids**: Never expose numeric `id` columns in URLs or APIs; use `refid` to prevent ID enumeration.
- **Verify Ownership**: When build reseller dashboards, always scope queries with `owner_id` to prevent cross-tenant access.
- **Sanitize Metadata**: Avoid storing sensitive credentials or raw PII in the `metadata` JSON field without encryption.
