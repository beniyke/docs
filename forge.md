# Forge

**Forge** is a package for generating, activating, and verifying software licenses allowing you to transform products into unique, secure, and verifiable cryptographic keys for distribution to your clients.

## Features

- **Automated Key Generation**: Secure, non-sequential license key generation with custom formatting.
- **Duration-Based Expiry**: Flexible validity periods (days) that window after the point of activation.
- **Activation Lifecycle**: Multi-stage state machine (Pending -> Active -> Expired/Revoked).
- **Client Binding**: Securely bind licenses to `Client` entities to prevent unauthorized usage.
- **Verification Engine**: Ultra-fast lookup and status verification for application-side checks.
- **Product Integration**: Seamlessly links licenses to **Wave** products to inheritance metadata and weights.

## Installation

Forge is a **package** that requires installation to set up the secure forging environment.

### Install the Package

```bash
php dock package:install Forge --packages
```

This command will automatically:

- Create necessary database tables `forge_*`.
- Register the `ForgeServiceProvider`.

## Architecture & Lifecycle

Forge operates as a stateful engine where every key follows a strict lifecycle.

### The Minting Layer

#### Generation

When a license is first created via `Forge::make()`, it is "Minted". It is assigned a unique `key` and a `product_id`, but it remains in a `Pending` state. It is not yet valid for usage.

### The Bonding Layer

#### Activation

A license is "Bonded" when it is assigned to a `Client`. At this moment, the `activated_at` timestamp is set, and the `expires_at` date is calculated based on the configured `duration_days`. The status transitions to `Active`.

### The Enforcement Layer

#### Verification & Revocation

Applications call `Forge::verify($key)` to check if a license is still in an authorized state. If an administrator manually cancels a license, it is "Revoked", immediately failing all future verification checks. Methods like `activate` and `revoke` accept both IDs and Model objects.

## Package Integrations

Forge is a core component of the Anchor ecosystem and follows strict **Service-Oriented Architecture (SOA)** principles. Interactions with other packages occur exclusively via Facades to maintain clean boundaries.

### Integration with Wave

#### Product Catalog

Forge acts as the licensing layer for commercial products defined in the **Wave** package. Every license must correspond to a valid `Wave\Models\Product`.

- **Logic**: Forge use the `Wave` facade to resolve product details before minting.
- **Decoupling**: The `Licence` model stores a `product_id` but does not possess a direct relationship to the Wave model. This prevents database migrations in Wave from breaking Forge.

```php
use Wave\Wave;
use Forge\Forge;

// Fetch a product via the Wave Facade
$product = Wave::findProduct('SaaS-Premium');

// Pass the product (or just its ID) to the builder
$licence = Forge::make()
    ->product($product)
    ->create();
```

### Integration with Client

#### Identity & Bonding

Licenses are useless until they are "Bonded" to a **Client**. This relationship defines who is authorized to use the cryptographic key.

- **Validation**: During activation, Forge calls `Client::find($id)` to ensure the recipient is a valid entity.
- **Notification Integration**: When a license is activated, Forge resolves the Client's email and name via the `Client` facade to dispatch activation alerts.

```php
use Client\Client;
use Forge\Forge;

// Find the client via the Client Facade
$client = Client::findByEmail('user@example.com');

// Activate and bind the license to this client
Forge::activate($licenceId, $client);
```

### Integration with Wallet

#### E-Commerce

While Forge manages the status of the license, the **Wallet** or **Wave Invoice** layers typically handle the financial transaction. Once a payment is successful, systems usually call `Forge::make()` as a post-purchase action.

## Core Concepts

Forge distinguishes between the "Agreement" and the "Access":

- **License Keys**: The actual string provided to the client.
- **Duration**: The number of days the key is valid _after_ activation. Set this to `null` for lifetime licenses.

## Basic Usage

### License Generation

Forge generates licenses for products managed by the **Wave** package. You typically retrieve the product first to ensure it exists.

```php
use Forge\Forge;
use Wave\Wave;

// 1. Find the product from the Wave commercial catalog
$product = Wave::findProduct('PLATFORM-PRO'); // Find the product by refid

// 2. Mint a new license for this specific product
$licence = Forge::make()
    ->product($product) // Accepts Product object or ID
    ->duration(365)     // Valid for 1 year after activation
    ->create();
```

### Activation Flow

Bind a minted license to a specific client. Forge methods accept both IDs and Model objects.

```php
use Forge\Forge;

// Using objects
$result = Forge::activate($licence, $client);

// OR using IDs
$result = Forge::activate($licence->id, $client->id);

if ($result) {
    // License is now Active and expires 365 days from now
}
```

### Verification

Check license validity in your local or remote application.

```php
use Forge\Forge;

$validLicence = Forge::verify($inputKey);

if ($validLicence) {
    // Access granted
}
```

### Query Scopes

The `Licence` model includes fluent scopes for status and lifecycle management.

```php
use Forge\Models\Licence;

// Get all currently active and valid licenses
$active = Licence::active()->get();

// Get licenses pending activation
$pending = Licence::pending()->get();

// Get expired or past-due licenses
$expired = Licence::expired()->get();

// Get revoked licenses
$revoked = Licence::revoked()->get();

// Find licenses expiring in the next 7 days
$expiring = Licence::expiringIn(7)->get();
```

## Analytics & Business Intelligence

The Forge facade provides access to a dedicated `AnalyticsManager` for tracking license distribution trends and forecasting expirations.

### Distribution Overview

Retrieve a breakdown of licenses by their current status (Minted, Active, Revoked, etc.).

```php
$stats = Forge::analytics()->mintingStats();

/*
[
    'total' => 150,
    'status' => [
        'pending' => 20,
        'active' => 120,
        'revoked' => 10
    ]
]
*/
```

### Expiration Forecasting

Predict how many active licenses will expire within a given window. This is critical for churn prevention and renewal outreach.

```php
// Count licenses expiring in the next 30 days
$churnRisk = Forge::analytics()->expirationForecast(30);
```

### Trending & Historical Data

Forge provides a fluent, chainable API for time-series data optimized for line charts.

#### Minting Trends

Track how many licenses are generated over time. You can chain interval methods (`daily()`, `monthly()`, `yearly()`) before the trend call.

```php
// Monthly trends for the entire year
$trends = Forge::analytics()->monthly()->mintingTrends('2023-01-01', '2023-12-31');

// Returns ['2023-01' => 120, '2023-02' => 145, ...]
```

#### Scoped Analytics

You can scope any analytical call to a specific **Client** or **Reseller** (Owner). This is perfect for building personalized dashboards.

```php
// Track activations for a specific reseller's customers
$resellerActivity = Forge::analytics()
    ->forReseller($resellerId)
    ->daily()
    ->activationTrends('2023-10-01', '2023-10-14');

// Get minting stats for a specific client
$clientStats = Forge::analytics()
    ->forClient($clientId)
    ->mintingStats();
```

### Product Popularity

Identify which commercial products are driving the most license generation.

```php
$popularity = Forge::analytics()->productPopularity();

// Returns [product_id => count]
```

## Service API Reference

### Forge Facade

The `Forge\Forge` facade is the unified entry point for all licensing operations.

| Method                   | Description                                  |
| :----------------------- | :------------------------------------------- |
| `make()`                 | Returns a fluent `LicenceBuilder`.           |
| `activate(id, clientId)` | Binds and activates a license for a client.  |
| `verify(key)`            | Validates key existence and active status.   |
| `findByRefid(refid)`     | Find a license by its public reference ID.   |
| `revoke(id)`             | Manually invalidate a license key.           |
| `analytics()`            | Access the `AnalyticsManager` for reporting. |

### AnalyticsManager Reference

| Method                                   | Description                                      |
| :--------------------------------------- | :----------------------------------------------- |
| `mintingStats(start, end)`               | Aggregate totals by status.                      |
| `expirationForecast(days)`               | Predict churn for a future window.               |
| `productPopularity(start, end)`          | Identify top-performing commercial products.     |
| `mintingTrends(start, end, interval)`    | Daily/Monthly time-series of license generation. |
| `activationTrends(start, end, interval)` | Daily/Monthly time-series of bonds to clients.   |
