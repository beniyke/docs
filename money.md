# Money

The Money package provides an immutable, high-precision solution for financial arithmetic within the Anchor Framework. By leveraging arbitrary precision math (BCMath), it eliminates floating-point errors and ensures absolute reliability for monetary operations.

## Key Features

- Immutable money objects
- BCMath precision (no float errors)
- 30 major currencies with ISO 4217 support
- Full arithmetic operations
- Percentage calculations
- Fair money allocation
- Multiple formatting options
- Comprehensive edge case handling

## Requirements

- PHP 8.1+
- BCMath extension

## Installation

Money is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Money --packages
```

This will automatically:

- Register the `MoneyServiceProvider`
- Enable global helper functions (`money()`, `money_minor()`)

## Quick Start

```php
use Money\Money;

// Create money
$price = Money::make(10000, 'USD'); // $100.00

// Arithmetic
$total = $price->add(Money::make(2000, 'USD')); // $120.00
$discounted = $price->subtractPercentage(10);    // $90.00

// Formatting
echo $price->formatSimple(); // "$100.00"

// Comparison
if ($total->greaterThan($price)) {
    echo "Total is greater!";
}
```

## Creating Money

### From Minor Units (Cents)

```php
// PRIMARY METHOD - Most fluent
$money = Money::make(10000, 'USD'); // $100.00

// ALIASES - All equivalent
$money = Money::create(10000, 'USD');
$money = Money::from(10000, 'USD');
$money = Money::cents(10000, 'USD');
```

### From Major Units (Dollars)

```php
// PRIMARY METHOD - Most fluent
$money = Money::amount(100.50, 'USD');  // $100.50

// CURRENCY-SPECIFIC - Intuitive
$money = Money::dollars(100.50);        // $100.50 (USD)
$money = Money::euros(75.25);           // €75.25
$money = Money::pounds(50.00);          // £50.00
$money = Money::yen(1000);              // ¥1000
$money = Money::yuan(500);              // ¥500 (CNY)
$money = Money::rupees(750);            // ₹750
$money = Money::reals(250);             // R$250 (BRL)
$money = Money::pesos(300);             // MX$300
$money = Money::rand(150);              // R150 (ZAR)
$money = Money::rubles(400);            // ₽400
$money = Money::won(50000);             // ₩50000
$money = Money::francs(200);            // CHF200
$money = Money::krona(350);             // kr350 (SEK)
```

### Special Cases

```php
// Zero money
$zero = Money::zero('USD');

// From array
$money = Money::fromArray([
    'amount' => 10000,
    'currency' => 'USD'
]);
```

## Global Helpers

The Money package provides convenient global helpers for creating Money instances.

```php
// Create money from major units (e.g., dollars)
$money = money(100.50, 'USD');

// Create money from minor units (e.g., cents)
$money = money_minor(1050, 'USD'); // $10.50
```

## Arithmetic Operations

All operations return **new instances** (immutable).

### Addition

```php
$a = Money::make(100, 'USD');
$b = Money::make(200, 'USD');

$total = $a->add($b); // $3.00

// Add multiple
$total = $a->add($b, $c, $d);
```

### Subtraction

```php
$price = Money::make(500, 'USD');
$discount = Money::make(100, 'USD');

$final = $price->subtract($discount); // $4.00
```

### Multiplication

```php
use Money\RoundingMode;

$price = Money::make(100, 'USD');

$doubled = $price->multiply(2);    // $2.00
$result = $price->multiply(1.5, RoundingMode::HALF_UP); // $1.50
```

### Division

```php
$total = Money::make(300, 'USD');

$perPerson = $total->divide(3); // $1.00
```

### Other Operations

```php
// Modulo
$money = Money::make(100, 'USD');
$remainder = $money->mod(Money::make(30, 'USD')); // $0.10

// Absolute value
$negative = Money::make(-100, 'USD');
$positive = $negative->absolute(); // $1.00

// Negate
$money = Money::make(100, 'USD');
$negative = $money->negative(); // -$1.00
```

## Percentage Operations

### Calculate Percentage

```php
$price = Money::make(10000, 'USD'); // $100

$tax = $price->percentage(15); // $15 (15% of $100)
```

### Add Percentage

```php
$price = Money::make(10000, 'USD');

$withMarkup = $price->addPercentage(20); // $120 (+20%)
```

### Subtract Percentage

```php
$price = Money::make(10000, 'USD');

$withDiscount = $price->subtractPercentage(10); // $90 (-10%)
```

### Get Ratio

```php
$a = Money::make(200, 'USD');
$b = Money::make(100, 'USD');

$ratio = $a->ratioOf($b); // 200.0 (200%)
```

## Allocation

Split money fairly with automatic remainder distribution.

### Allocate by Ratios

```php
$money = Money::make(100, 'USD'); // $1.00

// Split 1:2:1 ratio
$parts = $money->allocate([1, 2, 1]);
// [$0.25, $0.50, $0.25]
```

### Allocate Equally

```php
$money = Money::make(100, 'USD');

$parts = $money->allocateTo(3);
// [$0.34, $0.33, $0.33] - remainder distributed to first parts
```

### Fair Allocation

```php
// Split bill among friends
$bill = Money::dollars(127.50);
$perPerson = $bill->allocateTo(5);
// Each person pays: $25.50, $25.50, $25.50, $25.50, $25.50
```

## Comparison

### Equality

```php
$a = Money::make(100, 'USD');
$b = Money::make(100, 'USD');

$a->equals($b); // true
```

### Greater/Less Than

```php
$a = Money::make(100, 'USD');
$b = Money::make(200, 'USD');

$a->lessThan($b);            // true
$b->greaterThan($a);         // true
$a->lessThanOrEqual($b);     // true
$b->greaterThanOrEqual($a);  // true
```

### Compare

```php
$a = Money::make(100, 'USD');
$b = Money::make(200, 'USD');

$a->compare($b); // -1 (less than)
$b->compare($a); // 1  (greater than)
$a->compare($a); // 0  (equal)
```

### State Checks

```php
$money = Money::make(100, 'USD');

$money->isZero();      // false
$money->isPositive();  // true
$money->isNegative();  // false

// Check currency
$money->isSameCurrency(Money::make(50, 'USD')); // true

// Check divisibility
$money->isDivisibleBy(10); // true
```

## Currency Conversion

Convert money between currencies using exchange rates.

### Basic Conversion

```php
use Money\CurrencyConverter;
use Money\Providers\FixedExchangeRateProvider;

// Create provider and set rates
$provider = new FixedExchangeRateProvider();
$provider->setRate('USD', 'EUR', 0.85);
$provider->setRate('USD', 'GBP', 0.73);

// Create converter
$converter = new CurrencyConverter($provider);

// Convert money
$usd = Money::dollars(100);
$eur = $usd->convertTo('EUR', $converter); // €85.00
```

### Chaining Conversions

```php
$usd = Money::dollars(100);
$eur = $usd->convertTo('EUR', $converter);
$gbp = $eur->convertTo('GBP', $converter);
```

### Inverse Rates

The provider automatically calculates inverse rates:

```php
$provider->setRate('USD', 'EUR', 0.85);

// Both work:
$eur = $usd->convertTo('EUR', $converter); // Uses direct rate
$usd = $eur->convertTo('USD', $converter); // Uses inverse (1/0.85)
```

**Example**

```php
// E-commerce with multi-currency support
$provider = new FixedExchangeRateProvider();
$provider
    ->setRate('USD', 'EUR', 0.85)
    ->setRate('USD', 'GBP', 0.73)
    ->setRate('USD', 'JPY', 110.0);

$converter = new CurrencyConverter($provider);

// Product price in USD
$priceUsd = Money::dollars(99.99);

// Show in customer's currency
$priceEur = $priceUsd->convertTo('EUR', $converter);
echo $priceEur->formatSimple(); // "€84.99"
```

### Multi-Currency Shopping Cart

```php
$provider = new FixedExchangeRateProvider();
$provider->setRate('USD', 'EUR', 0.85);
$converter = new CurrencyConverter($provider);

// Items in different currencies
$item1 = Money::dollars(29.99);
$item2 = Money::euros(25.00);
$item3 = Money::dollars(49.99);

// Convert all to USD
$total = $item1
    ->add($item2->convertTo('USD', $converter))
    ->add($item3);

echo $total->formatSimple(); // Total in USD
```

## Fluent Rounding

Convenient fluent methods for all rounding scenarios.

### All Fluent Rounding Methods

```php
$money = Money::make(100, 'USD');

// Ceiling - always round up
$result = $money->multiplyAndRoundUp(1.5);      // $1.50 → $2.00

// Floor - always round down
$result = $money->multiplyAndRoundDown(1.5);    // $1.50 → $1.00

// Half Up - round .5 up (most common)
$result = $money->multiplyAndRoundHalfUp(1.5);  // $1.50 → $2.00

// Half Down - round .5 down
$result = $money->multiplyAndRoundHalfDown(1.5); // $1.50 → $1.00

// Half Even - banker's rounding (round to nearest even)
$result = $money->multiplyAndRoundHalfEven(1.5); // $1.50 → $2.00

// Short aliases
$result = $money->roundUp(1.5);    // Same as multiplyAndRoundUp
$result = $money->roundDown(1.5);  // Same as multiplyAndRoundDown
```

### Rounding Mode Comparison

|Method|1.4|1.5|1.6|Description|
|:---|:---|:---|:---|:---|
|`multiplyAndRoundUp()`|2|2|2|Always round up (ceiling)|
|`multiplyAndRoundDown()`|1|1|1|Always round down (floor)|
|`multiplyAndRoundHalfUp()`|1|2|2|Round .5 up (default)|
|`multiplyAndRoundHalfDown()`|1|1|2|Round .5 down|
|`multiplyAndRoundHalfEven()`|1|2|2|Banker's rounding|

### Example: Tax and Rounds

```php
// Tax calculation (conservative - round up)
$price = Money::dollars(99.99);
$tax = $price->multiplyAndRoundUp(0.0825);

// Discount calculation (customer-friendly - round down)
$discount = $price->multiplyAndRoundDown(0.15);

// Financial calculations (banker's rounding)
$interest = $balance->multiplyAndRoundHalfEven(0.05);
```

## Formatting

### Simple Formatting

```php
$money = Money::make(123456, 'USD');

$money->formatSimple(); // "$1,234.56"
```

### Custom Decimals

```php
$money = Money::make(123456, 'USD');

$money->formatByDecimal(2); // "$1,234.56"
$money->formatByDecimal(0); // "$1,235"
```

### Locale-Aware (requires Intl extension)

```php
$money = Money::make(123456, 'USD');

$money->format('en_US'); // "$1,234.56"
$money->format('de_DE'); // "1.234,56 $"
```

### Raw Amount

```php
$money = Money::make(123456, 'USD');

$money->formatWithoutSymbol(); // "1,234.56"
```

## Accessors

Retrieve the raw values from the Money object.

```php
$money = Money::make(1050, 'USD'); // $10.50

// Get minor units (int) - Default storage format
$cents = $money->getAmount();      // 1050
$cents = $money->getMinorAmount(); // 1050 (alias)

// Get major units (float) - Careful with float precision!
$dollars = $money->getMajorAmount(); // 10.5

// Get currency object
$currency = $money->getCurrency();
echo $currency->getCode(); // "USD"
```

## Serialization

### To Array

```php
$money = Money::make(10000, 'USD');

$array = $money->toArray();
// [
//     'amount' => 10000,
//     'currency' => 'USD',
//     'formatted' => '$100.00'
// ]
```

### JSON

```php
$money = Money::make(10000, 'USD');

$json = json_encode($money);
// {"amount":10000,"currency":"USD","formatted":"$100.00"}
```

### String

```php
$money = Money::make(10000, 'USD');

echo $money->toString();  // "$100.00"
echo (string) $money;     // "$100.00"
```

### Database

```php
$money = Money::make(10000, 'USD');

$dbValue = $money->toDatabaseValue(); // 10000 (integer)
```

## Aggregation

### Sum

```php
$prices = [
    Money::make(100, 'USD'),
    Money::make(200, 'USD'),
    Money::make(300, 'USD'),
];

$total = Money::sum($prices); // $6.00
```

### Average

```php
$prices = [
    Money::make(100, 'USD'),
    Money::make(200, 'USD'),
    Money::make(300, 'USD'),
];

$avg = Money::average($prices); // $2.00
```

### Min/Max

```php
$min = Money::min(
    Money::make(300, 'USD'),
    Money::make(100, 'USD'),
    Money::make(200, 'USD')
); // $1.00

$max = Money::max(
    Money::make(100, 'USD'),
    Money::make(300, 'USD'),
    Money::make(200, 'USD')
); // $3.00
```

## Supported Currencies

30 major world currencies:

|Code|Name|Symbol|
|:---|:---|:---|
|USD|US Dollar|$|
|EUR|Euro|€|
|GBP|British Pound|£|
|JPY|Japanese Yen|¥|
|CNY|Chinese Yuan|¥|
|CAD|Canadian Dollar|C$|
|AUD|Australian Dollar|A$|
|CHF|Swiss Franc|CHF|
|INR|Indian Rupee|₹|
|BRL|Brazilian Real|R$|
|MXN|Mexican Peso|MX$|
|ZAR|South African Rand|R|
|RUB|Russian Ruble|₽|
|KRW|South Korean Won|₩|
|SGD|Singapore Dollar|S$|
|HKD|Hong Kong Dollar|HK$|
|SEK|Swedish Krona|kr|
|NOK|Norwegian Krone|kr|
|DKK|Danish Krone|kr|
|PLN|Polish Zloty|zł|
|THB|Thai Baht|฿|
|IDR|Indonesian Rupiah|Rp|
|MYR|Malaysian Ringgit|RM|
|PHP|Philippine Peso|₱|
|NZD|New Zealand Dollar|NZ$|
|TRY|Turkish Lira|₺|
|AED|UAE Dirham|د.إ|
|SAR|Saudi Riyal|ر.س|
|NGN|Nigerian Naira|₦|
|EGP|Egyptian Pound|E£|

## Edge Cases

### Currency Mismatch

```php
try {
    $usd = Money::make(100, 'USD');
    $eur = Money::make(50, 'EUR');
    $usd->add($eur); // Throws CurrencyMismatchException
} catch (CurrencyMismatchException $e) {
    // Handle error
}
```

### Division by Zero / Invalid Amounts

```php
try {
    $money = Money::make('invalid', 'USD');
    $money->divide(0);
} catch (InvalidAmountException $e) {
    // Handle error
}
```

### Invalid Currency

```php
try {
    $money = Money::make(100, 'XYZ'); // Throws InvalidCurrencyException
} catch (InvalidCurrencyException $e) {
    // Handle error
}
```

### Missing Exchange Rate

```php
try {
    $money->convertTo('JPY', $converter); // Throws ExchangeRateNotFoundException if rate missing
} catch (ExchangeRateNotFoundException $e) {
    // Handle error
}
```

### Large Numbers

```php
// Uses BCMath - safe for very large numbers
$huge = Money::make('999999999999999999', 'USD');
$result = $huge->multiply(2); // Works correctly
```

### Negative Amounts

```php
$negative = Money::make(-100, 'USD');

$negative->isNegative();  // true
$negative->absolute();    // Money::make(100, 'USD')
```

### Allocation Remainders

```php
$money = Money::make(10, 'USD');
$parts = $money->allocateTo(3);
// [$0.04, $0.03, $0.03] - remainder distributed fairly
```

## Best Practices

### Always Store as Minor Units

```php
// ✅ Good - Store cents in database
$product->price = 10000; // $100.00

// ❌ Bad - Don't store floats
$product->price = 100.00;
```

### Use Money Objects Throughout

```php
// ✅ Good
public function calculateTotal(Money $price, int $quantity): Money
{
    return $price->multiply($quantity);
}

// ❌ Bad
public function calculateTotal(float $price, int $quantity): float
{
    return $price * $quantity; // Precision loss!
}
```

### Handle Currency Mismatches

```php
// ✅ Good
if (!$price->isSameCurrency($tax)) {
    throw new Exception('Currency mismatch');
}
$total = $price->add($tax);

// ❌ Bad
$total = $price->add($tax); // May throw unexpectedly
```

### Use Allocation for Splitting

```php
// ✅ Good - Fair distribution
$parts = $total->allocateTo(3);

// ❌ Bad - May lose cents
$perPerson = $total->divide(3);
```

### Format Only for Display

```php
// ✅ Good
echo $price->formatSimple(); // Display only

// ❌ Bad
$stored = $price->formatSimple(); // Don't store formatted strings
```

## Model Integration

```php
class Product extends BaseModel
{
    protected $casts = [
        'price' => 'integer', // Store as cents
    ];

    public function getPriceAttribute($value): Money
    {
        return Money::make($value, 'USD');
    }

    public function setPriceAttribute(Money $money): void
    {
        $this->attributes['price'] = $money->toDatabaseValue();
    }
}

// Usage
$product = new Product();
$product->price = Money::dollars(99.99);
$product->save();

echo $product->price->formatSimple(); // "$99.99"
```
