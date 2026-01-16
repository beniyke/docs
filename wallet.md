# Wallet

The Wallet provides a production-ready digital wallet system with double-entry ledger accounting, ensuring financial integrity through atomic transactions, immutable records, and automatic balance reconciliation.

## Features

- **Double-Entry Ledger**: Immutable transaction history ensuring data integrity
- **Atomic Operations**: Row-level locking protects against race conditions
- **Multi-Currency**: Support for multiple currencies per user (USD, EUR, etc.)
- **Idempotency**: Prevents duplicate transactions using reference IDs
- **Fee Management**: Configurable fees (Fixed, Percentage, Tiered)
- **Model Integration**: "HasWallet" trait for seamless Eloquent integration
- **Audit Trail**: Full history of every credit, debit, transfer, and refund
- **Reconciliation**: Automated balance verification and correction logic
- **Analytics**: Dashboard-ready reporting with daily/monthly volumes, summaries, and charts

## Installation

Wallet is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Wallet --packages
```

This will automatically:

- Run database migrations `wallet_*` tables
- Register the service provider
- Publish the configuration file

### Configuration

Configuration file: `App/Config/wallet.php`

```php
return [
    // Default currency for new wallets
    'default_currency' => env('WALLET_DEFAULT_CURRENCY', 'USD'),

    // Automatically create wallet on first transaction
    'auto_create' => env('WALLET_AUTO_CREATE', true),

    // Transaction limits (in smallest unit - cents)
    'limits' => [
        'credit' => [
            'min' => env('WALLET_CREDIT_MIN', 100), // $1.00
            'max' => env('WALLET_CREDIT_MAX', null),
        ],
        'debit' => [
            'min' => env('WALLET_DEBIT_MIN', 100),
            'max' => env('WALLET_DEBIT_MAX', null),
        ],
    ],

    // Fee configuration
    'fees' => [
        'credit' => [
            'enabled' => false,
            'type' => 'PERCENTAGE', // FIXED, PERCENTAGE, TIERED
            'amount' => 0,
            'percentage' => 0.029, // 2.9%
        ],
        'debit' => [
            'enabled' => false,
            'type' => 'FIXED',
            'amount' => 200, // $2.00
        ],
    ],

    // Reconciliation settings
    'reconciliation' => [
        'auto_fix' => env('WALLET_AUTO_FIX_BALANCE', true),
        'notify_on_mismatch' => env('WALLET_NOTIFY_MISMATCH', true),
    ],
];
```

Environment variables (`.env`):

```env
WALLET_DEFAULT_CURRENCY=USD
WALLET_AUTO_CREATE=true
WALLET_LOGGING=true
WALLET_AUTO_FIX_BALANCE=true
```

## Basic Usage

### Static Facade

Use the `Wallet` facade for quick, static access to wallet operations:

```php
use Wallet\Wallet;
use Money\Money;

// Credit a wallet (Fluent) (Use integer/float for dollars/major unit)
Wallet::transaction($walletId)
    ->credit(100, 'USD') // $100.00
    ->description('Top-up funds')
    ->execute();

// Debit a wallet (Fluent)
Wallet::transaction($walletId)
    ->debit(25, 'USD') // $25.00
    ->description('Purchase')
    ->execute();

// Check balance
$balance = Wallet::getBalance($walletId);
echo $balance->formatSimple(); // "$75.00"

// Transfer funds between wallets (Fluent)
Wallet::transfer($senderWalletId, $receiverWalletId, 50, 'USD'); // $50.00
```

## Model Integration

#### Recommended

Add the `HasWallet` trait to any user or business model:

```php
use Wallet\Traits\HasWallet;

class User extends BaseModel
{
    use HasWallet;
}
```

### Usage with Models

```php
use Wallet\Exceptions\InsufficientFundsException;

$user = User::find(1);

// Get or create wallet (USD by default)
$wallet = $user->getOrCreateWallet('USD');

// Credit: Add funds (Fluent)
$user->transaction('USD')
    ->credit(100) // $100.00 USD
    ->description('Top-up via Stripe')
    ->processor('stripe', 'ch_123456')
    ->execute();

// Debit: Spend funds (Fluent)
try {
    $user->transaction('USD')
        ->debit(50) // $50.00 USD
        ->description('Purchase Order #99')
        ->execute();
} catch (InsufficientFundsException $e) {
    // Handle low balance
}

// Check Balance (Fluent)
if ($user->canAfford(50, 'USD')) { // $50.00 USD
    // Proceed...
}
```

## Use Case Walkthrough

This section demonstrates the complete wallet lifecycle in a typical SaaS or e-commerce application.

#### Scenario: User Registration to First Purchase

**Setup: Add HasWallet to User Model**

```php
// App/Models/User.php
use Wallet\Traits\HasWallet;

class User extends BaseModel
{
    use HasWallet;
}
```

**User Signs Up → Wallet is Created Automatically**

When a user registers, you can create their wallet immediately or let it be created on first transaction:

```php
// In your registration controller
public function register(): Response
{
    $user = User::create([
        'name' => $this->request->post('name'),
        'email' => $this->request->post('email'),
        'password' => bcrypt($this->request->post('password')),
    ]);

    // Option A: Create wallet immediately
    $wallet = $user->getOrCreateWallet('USD');

    // Option B: Skip this - wallet will be auto-created on first transaction

    return $this->response->json([
        'user' => $user,
        'wallet_id' => $wallet->id,
        'balance' => '$0.00'
    ]);
}
```

**3. User Funds Wallet via Payment Gateway**

When the user wants to add money, you integrate with the `Pay` package:

```php
// FundingController.php
use Pay\Pay;

public function initiateFunding(): Response
{
    $user = $this->auth->user();
    $amount = $this->request->post('amount'); // e.g., 50 for $50.00
    $wallet = $user->getOrCreateWallet('USD');

    // Initialize payment with Pay
    $payment = Pay::amount($amount)
        ->email($user->email)
        ->reference('fund_' . uniqid())
        ->metadata([
            'wallet_id' => $wallet->id,
            'intention' => 'fund',
            'user_id' => $user->id,
        ])
        ->initialize();

    // Return authorization URL for redirect
    return $this->response->json([
        'authorization_url' => $payment->authorization_url,
        'reference' => $payment->reference,
    ]);
}
```

**Webhook: Payment Confirmed → Wallet Credited**

The `WalletFundingListener` in the Wallet package listens for `PaymentSuccessful` events and credits the wallet automatically:

- This happens automatically via the listener
- When webhook confirms payment:

  - PaymentSuccessful event is dispatched by Pay package
  - WalletFundingListener receives it
  - Wallet is credited with the paid amount

- User's wallet now shows $50.00

**User Makes a Purchase**

Now the user can spend their wallet balance:

```php
// CheckoutController.php
use Wallet\Exceptions\InsufficientFundsException;

public function checkout(): Response
{
    $user = $this->auth->user();
    $orderTotal = 29.99; // $29.99

    // Check if user can afford
    if (!$user->canAfford($orderTotal, 'USD')) {
        return $this->response->json([
            'error' => 'Insufficient funds',
            'balance' => $user->getBalance('USD')->formatSimple(),
            'required' => '$' . number_format($orderTotal, 2),
        ], 402);
    }

    try {
        // Debit the wallet
        $transaction = $user->transaction('USD')
            ->debit($orderTotal)
            ->description('Order #' . $this->request->post('order_id'))
            ->meta([
                'order_id' => $this->request->post('order_id'),
                'product_ids' => $this->request->post('items'),
            ])
            ->execute();

        // Mark order as paid...
        Order::find($this->request->post('order_id'))->markPaid($transaction->reference_id);

        return $this->response->json([
            'status' => 'success',
            'transaction_id' => $transaction->reference_id,
            'new_balance' => $user->getBalance('USD')->formatSimple(), // "$20.01"
        ]);

    } catch (InsufficientFundsException $e) {
        return $this->response->json(['error' => 'Payment failed'], 402);
    }
}
```

**Admin Issues a Refund**

If the order is cancelled, you can refund the transaction:

```php
// Admin/RefundController.php
use Wallet\Wallet;

public function refundOrder(): Response
{
    $order = Order::find($this->request->post('order_id'));

    // Full refund
    $refundTx = Wallet::refund($order->transaction_reference);

    // Or partial refund (e.g., $10 of the original $29.99)
    $refundTx = Wallet::startRefund($order->transaction_reference)
        ->amount(10)
        ->execute();

    return $this->response->json([
        'refund_id' => $refundTx->reference_id,
        'refunded_amount' => $refundTx->getAmountAsMoney()->formatSimple(),
    ]);
}
```

**Peer-to-Peer Transfer (Optional)**

If your app supports sending money between users:

```php
// TransferController.php
public function send(): Response
{
    $sender = $this->auth->user();
    $recipient = User::find($this->request->post('recipient_id'));
    $amount = $this->request->post('amount'); // e.g., 15 for $15.00

    if (!$sender->canAfford($amount, 'USD')) {
        return $this->response->json(['error' => 'Insufficient funds'], 402);
    }

    // Execute transfer
    $result = $sender->transfer($amount)
        ->to($recipient)
        ->description('Sent to ' . $recipient->name)
        ->execute();

    return $this->response->json([
        'status' => 'success',
        'sender_balance' => $sender->getBalance('USD')->formatSimple(),
        'message' => "Sent \${$amount} to {$recipient->name}",
    ]);
}
```

## Implementation

### E-commerce Purchase Flow

```php
use App\Core\BaseController;
use Helpers\Http\Response;
use Money\Money;
use Wallet\Exceptions\InsufficientFundsException;
use Wallet\Exceptions\InvalidAmountException;

class OrderController extends BaseController
{
    public function checkout(): Response
    {
        // 1. Validate Input (Simple check for example)
        if (!$this->request->filled(['total_cents', 'order_id'])) {
            return $this->response->json(['error' => 'Missing fields'], 400);
        }

        // 2. Get Authenticated User
        $user = $this->auth->user();
        $total = Money::make($this->request->post('total_cents'), 'USD');

        try {
            // 3. Atomic debit operation (Fluent)
            $transaction = $user->transaction('USD')
                ->debit($total)
                ->description("Order #{$this->request->post('order_id')}")
                ->meta([
                    'order_id' => $this->request->post('order_id'),
                    'items' => $this->request->post('items_count'),
                ])
                ->execute();

            // 4. Return Success Response
            return $this->response->json([
                'status' => 'success',
                'new_balance' => $user->getBalance()->formatSimple(),
                'transaction_id' => $transaction->id
            ]);

        } catch (InsufficientFundsException $e) {
            return $this->response->json(['error' => 'Insufficient funds'], 402);
        } catch (InvalidAmountException $e) {
            return $this->response->json(['error' => 'Invalid amount'], 400);
        }
    }
}
```

### Peer-to-Peer Transfer

```php
// Transfer $50.00 from User A to User B
$sender = User::find(1);
$receiver = User::find(2);

$amount = Money::make(5000, 'USD');

if ($sender->canAfford($amount)) {
    // Fluent Transfer
    $sender->transfer($amount)
        ->to($receiver)
        ->description('Lunch money reimbursement')
        ->execute();
}
```

### Funding via Pay

You can fund a wallet using any gateway supported by the `Pay` package. The `WalletFundingListener` automatically credits the wallet when a `PaymentSuccessful` event is fired with the correct metadata.

```php
use Pay\Pay;

// Initialize payment with wallet_id and intention in metadata
Pay::amount(50) // $50.00
    ->email($user->email)
    ->reference('fund_' . uniqid())
    ->metadata([
        'wallet_id' => $wallet->id,
        'intention' => 'fund'
    ])
    ->initialize();

// Redirect user to payment gateway...
```

Upon successful payment (confirmed via webhook), the wallet will be credited automatically.

## Advanced Features

### Refunds

You can fully or partially refund any transaction. The system automatically links the refund to the original transaction.

```php
// Full Refund (Simple)
Wallet::refund('TXN_REFERENCE_ID_123');

// Partial Refund (Fluent)
// Refunds $10.00 of the original transaction
Wallet::startRefund('TXN_REFERENCE_ID_123')
    ->amount(10)
    ->execute();
```

### Fee Calculation

Fees can be configured globally in `Config/wallet.php` or defined via rules in the `wallet_fee_rules` table.

- **Types**: FIXED, PERCENTAGE, TIERED.
- **Example**: 2.9% + $0.30 per credit.

**Auto-calculate fees from config:**

```php
$user->transaction('USD')
    ->credit($amount)
    ->calculateFee() // Uses rules from wallet.php or DB
    ->execute();
```

**Specify a fixed fee manually:**

```php
$user->transaction('USD')
    ->credit(100)  // $100.00
    ->fee(2.50)    // $2.50 platform fee
    ->description('Deposit with platform fee')
    ->execute();
```

**Specify a percentage fee:**

```php
$user->transaction('USD')
    ->credit(100)       // $100.00
    ->feePercent(2.5)   // 2.5% = $2.50 fee
    ->description('Deposit with percentage fee')
    ->execute();
```

### Balance Reconciliation

The system includes a self-healing mechanism that verifies the cached `balance` against the immutable `wallet_transaction` ledger.

```php
// Run manual reconciliation
$isBalanced = Wallet::reconcile($walletId);
// Returns false (and fixes balance) if mismatch was found
```

### Analytics & Reporting

The `WalletAnalytics` service provides methods for dashboards, reports, and chart data.

**Access via Facade:**

```php
$analytics = Wallet::analytics();
```

**Get Wallet Summary:**

```php
$summary = Wallet::analytics()->getSummary($walletId, '2024-01-01', '2024-12-31');

// Returns:
// [
//     'wallet_id' => 1,
//     'currency' => 'USD',
//     'current_balance' => Money,
//     'total_credits' => Money,
//     'total_debits' => Money,
//     'transaction_count' => 150,
//     'credit_count' => 80,
//     'debit_count' => 70,
// ]
```

**Get Totals:**

```php
use Wallet\Enums\TransactionType;

// Total credits in date range
$totalCredits = Wallet::analytics()->getTotalCredits($walletId, '2024-01-01', '2024-12-31');

// Total debits
$totalDebits = Wallet::analytics()->getTotalDebits($walletId, '2024-01-01', '2024-12-31');

// Transaction count (using enum)
$count = Wallet::analytics()->getTransactionCount($walletId, TransactionType::CREDIT->value, '2024-01-01', '2024-12-31');
```

**Daily Volume for Charts:**

```php
$dailyData = Wallet::analytics()->getDailyVolume($walletId, '2024-12-01', '2024-12-31');

// Returns array with Money objects:
// [
//     ['date' => '2024-12-01', 'count' => 5, 'credits' => Money, 'debits' => Money, 'net' => Money],
//     ['date' => '2024-12-02', 'count' => 8, 'credits' => Money, 'debits' => Money, 'net' => Money],
// ]

foreach ($dailyData as $day) {
    echo "{$day['date']}: Credits {$day['credits']->format()}, Debits {$day['debits']->format()}";
}
```

**Monthly Volume:**

```php
$monthlyData = Wallet::analytics()->getMonthlyVolume($walletId, '2024-01-01', '2024-12-31');

// Returns array with Money objects:
// [
//     ['month' => '2024-01', 'count' => 45, 'credits' => Money, 'debits' => Money, 'net' => Money],
// ]

foreach ($monthlyData as $month) {
    echo "{$month['month']}: Net {$month['net']->format()}";
}
```

**Balance History (Running Balance):**

```php
$history = Wallet::analytics()->getBalanceHistory($walletId, '2024-12-01', '2024-12-31');

// Returns array with Money objects:
// [
//     ['date' => '2024-12-01', 'balance' => Money ($1,000.00)],
//     ['date' => '2024-12-02', 'balance' => Money ($1,250.00)],
// ]

foreach ($history as $day) {
    echo "{$day['date']}: {$day['balance']->format()}";
}
```

**Platform-Wide Analytics:**

```php
// For admin dashboards
$platformStats = Wallet::analytics()->getPlatformTotals('2024-01-01', '2024-12-31', 'USD');

// Returns Money objects for all monetary values:
// [
//     'total_wallets' => 1500,
//     'total_credits' => Money ($50,000.00),
//     'total_debits' => Money ($30,000.00),
//     'net_flow' => Money ($20,000.00),
// ]

echo "Net flow: {$platformStats['net_flow']->format()}";

// Top spenders
$topSpenders = Wallet::analytics()->getTopSpenders(10, '2024-01-01', '2024-12-31');
```

## Troubleshooting

| Error/Log                         | Cause                                                 | Solution                                                   |
| :-------------------------------- | :---------------------------------------------------- | :--------------------------------------------------------- |
| `CurrencyMismatchException`       | Attempting to credit/debit with a different currency. | Ensure `Money` instance matches the wallet currency.       |
| `InsufficientFundsException`      | Debit amount exceeds the available balance.           | Use `hasSufficientFunds()` check before debiting.          |
| "Balance reconciliation mismatch" | Manual DB edits or rare race conditions.              | Run the reconcile command or use `Wallet::reconcile($id)`. |

## Exception Handling

| Exception                       | Cause                                                 |
| ------------------------------- | ----------------------------------------------------- |
| `InsufficientFundsException`    | Debit amount exceeds balance.                         |
| `InvalidAmountException`        | Negative or zero amount provided.                     |
| `CurrencyMismatchException`     | Money currency differs from wallet currency.          |
| `DuplicateTransactionException` | Reused Reference ID or Idempotency Key.               |
| `WalletNotFoundException`       | Invalid Wallet ID.                                    |
| `TransactionNotFoundException`  | Invalid Reference ID for refund.                      |
| `PaymentProcessorException`     | An error occurred with an external payment processor. |

## Service API Reference

### Wallet (Facade)

| Method                                   | Description                                      |
| :--------------------------------------- | :----------------------------------------------- |
| `create($ownerId, $type, $currency)`     | Creates a new wallet for the given owner.        |
| `find(int $id)`                          | Retrieves a wallet by its internal primary key.  |
| `findByOwner($id, $type, $currency)`     | Locates a specific wallet for an owner.          |
| `credit(int $id, Money $m, array $meta)` | Adds funds to a wallet (transactional).          |
| `debit(int $id, Money $m, array $meta)`  | Removes funds from a wallet (checks balance).    |
| `transfer(int $f, int $t, Money $m)`     | Performs an atomic transfer between two wallets. |
| `startRefund(string $ref)`               | Starts a fluent refund builder.                  |
| `getBalance(int $id)`                    | Returns the current balance as a Money object.   |
| `transaction(int $id)`                   | Starts a fluent transaction builder.             |
| `analytics()`                            | Returns the `WalletAnalytics` service.           |

### WalletManager

| Method                                   | Description                                                                |
| :--------------------------------------- | :------------------------------------------------------------------------- |
| `refund(string $ref, ?Money $m = null)`  | Reverses a previous transaction (full or partial).                         |
| `reconcile(int $id)`                     | Audits transactions to ensure balance integrity.                           |
| `generateReferenceId()`                  | Generates a unique traceable reference for external links.                 |
| `getTransactionByReference(string $ref)` | Retrieves a transaction by its reference ID.                               |
| `toMoney(Transaction $tx, $field)`       | Converts a numeric transaction field (e.g. amount, fee) to a Money object. |
| `analytics()`                            | Returns the `WalletAnalytics` service.                                     |

### WalletAnalytics

| Method                                                | Description                                    |
| :---------------------------------------------------- | :--------------------------------------------- |
| `getTotalCredits(int $walletId, ?$from, ?$to)`        | Get total credits as `Money` for a date range. |
| `getTotalDebits(int $walletId, ?$from, ?$to)`         | Get total debits as `Money` for a date range.  |
| `getTotalByType(int $id, string $type, ?$from, ?$to)` | Get total by transaction type.                 |
| `getTransactionCount(int $id, ?$type, ?$from, ?$to)`  | Count transactions with optional filters.      |
| `getDailyVolume(int $id, $from, $to)`                 | Daily credits/debits for charts.               |
| `getMonthlyVolume(int $id, $from, $to)`               | Monthly credits/debits for charts.             |
| `getBalanceHistory(int $id, $from, $to)`              | Running balance over time for charts.          |
| `getSummary(int $id, ?$from, ?$to)`                   | Comprehensive wallet summary with all stats.   |
| `getTopSpenders(int $limit, ?$from, ?$to)`            | Top wallets by debit volume (platform-wide).   |
| `getPlatformTotals(?$from, ?$to)`                     | Platform-wide totals and wallet count.         |

### TransactionBuilder

Fluent builder for creating credits and debits. Obtained via `Wallet::transaction($id)` or `$model->transaction()`.

| Method                                   | Description                                          |
| :--------------------------------------- | :--------------------------------------------------- |
| `credit(int\|float\|Money $amount)`      | Set transaction as a credit with the given amount.   |
| `debit(int\|float\|Money $amount)`       | Set transaction as a debit with the given amount.    |
| `description(string $desc)`              | Add a human-readable description.                    |
| `meta(array $data)`                      | Merge additional metadata.                           |
| `with(string $key, mixed $value)`        | Add a single metadata key-value pair.                |
| `reference(string $refId)`               | Set a custom reference ID (for idempotency).         |
| `processor(string $name, ?string $txId)` | Record the payment processor and its transaction ID. |
| `fee(int\|float $amount)`                | Apply a fixed fee (in major currency units).         |
| `feePercent(int\|float $percent)`        | Apply a percentage fee (e.g., `2.5` for 2.5%).       |
| `calculateFee()`                         | Auto-calculate fee from config/DB rules.             |
| `execute()`                              | Execute the transaction and return `Transaction`.    |

### TransferBuilder

Fluent builder for wallet-to-wallet transfers. Obtained via `$model->transfer($amount)`.

| Method                           | Description                                           |
| :------------------------------- | :---------------------------------------------------- |
| `amount(int\|float\|Money $amt)` | Set the transfer amount.                              |
| `to(Wallet\|Model $recipient)`   | Set the destination (Wallet or model with trait).     |
| `description(string $desc)`      | Add a description for the transfer.                   |
| `meta(array $data)`              | Merge additional metadata.                            |
| `execute()`                      | Execute and return `['debit' => Tx, 'credit' => Tx]`. |

### RefundBuilder

Fluent builder for refunds. Obtained via `Wallet::startRefund($referenceId)`.

| Method                           | Description                                       |
| :------------------------------- | :------------------------------------------------ |
| `amount(int\|float\|Money $amt)` | Set partial refund amount (omit for full refund). |
| `execute()`                      | Execute and return the refund `Transaction`.      |

---

### Transaction Model Helpers

The `Transaction` model provides helper methods to access monetary values:

```php
$tx->getAmountAsMoney(): Money   // Gross transaction amount
$tx->getFeeAsMoney(): Money      // Fee charged
$tx->getNetAmountAsMoney(): Money // Net amount (amount - fee for credits)
```

### Enums

**`Wallet\Enums\TransactionType`**

| Value          | Description                            |
| :------------- | :------------------------------------- |
| `CREDIT`       | Funds added to wallet.                 |
| `DEBIT`        | Funds removed from wallet.             |
| `REFUND`       | Reversal of a previous debit.          |
| `TRANSFER_IN`  | Incoming transfer from another wallet. |
| `TRANSFER_OUT` | Outgoing transfer to another wallet.   |

**`Wallet\Enums\TransactionStatus`**

| Value       | Description                           |
| :---------- | :------------------------------------ |
| `PENDING`   | Transaction is awaiting confirmation. |
| `COMPLETED` | Transaction successfully processed.   |
| `FAILED`    | Transaction failed.                   |
| `REVERSED`  | Transaction was reversed/refunded.    |

## Console Commands

### View Balance

```bash
php dock wallet:balance <wallet_id>
```

### List Transactions

```bash
php dock wallet:transaction <wallet_id> --type=CREDIT --status=COMPLETED --limit=10
```

### Reconcile Wallet

```bash
php dock wallet:reconcile <wallet_id>
# Or reconcile all wallets:
php dock wallet:reconcile
```

## Security Best Practices

1.  **Row Locking**: The package uses `LockForUpdate` during transactions. Do not manually update wallet balances in the database; always use the API.
2.  **Idempotency**: Pass unique `idempotency_key` (or unique `reference_id`) for critical payments to prevent double-charging on retries.
3.  **Positive Verification**: The system throws `InvalidAmountException` if negative amounts are passed. Do not suppress this exception.
4.  **Logging**: Enable `WALLET_LOGGING` in production to audit all financial movements.

## API Quick Reference

#### `Wallet` Facade / `WalletManager`

Inject `Wallet\Services\WalletManager` for advanced usage. All facade calls are proxied to this service.

```php
Wallet::create(int|string $ownerId, string $ownerType, string $currency = 'USD'): Wallet
Wallet::find(int $walletId): ?Wallet
Wallet::findByOwner(int|string $ownerId, string $ownerType, string $currency = 'USD'): ?Wallet
Wallet::getBalance(int $walletId): Money
Wallet::credit(int $walletId, Money $amount, array $metadata = []): Transaction
Wallet::debit(int $walletId, Money $amount, array $metadata = []): Transaction
Wallet::transfer(int $fromId, int $toId, int|float|Money $amount, string|array $currency = 'USD', array $metadata = []): array
Wallet::refund(string $referenceId, ?Money $amount = null): Transaction
Wallet::reconcile(int $walletId): bool
Wallet::transaction(int $walletId): TransactionBuilder
Wallet::startRefund(string $referenceId): RefundBuilder
Wallet::analytics(): WalletAnalytics
```

#### `HasWallet` Trait

```php
$model->wallet(): MorphOne                                    // Wallet relationship
$model->createWallet(string $currency = 'USD'): Wallet        // Force create new wallet
$model->getOrCreateWallet(string $currency = 'USD'): Wallet   // Get or create wallet
$model->getBalance(string $currency = 'USD'): Money           // Get balance
$model->credit(Money $amount, array $metadata = []): Transaction
$model->debit(Money $amount, array $metadata = []): Transaction
$model->hasSufficientFunds(Money $amount): bool
$model->canAfford(int|float|Money $amount, string $currency = 'USD'): bool
$model->transactions(string $currency = 'USD'): ModelCollection
$model->transaction(string $currency = 'USD'): TransactionBuilder
$model->transfer(int|float|Money $amount, string $currency = 'USD'): TransferBuilder
```

## See Also

- [Pay](pay.md) - Payment gateway integration for wallet funding
- [Money](money.md) - Handling monetary amounts
- [Vault](vault.md) - For storing payment attachments
- [Core Globals](globals.md) - Global helper functions
