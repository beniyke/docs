# Number Helper

The `Helpers\Number\Number` class provides a robust set of utilities for formatting, converting, and performing mathematical operations on numbers.

> Use the `number()` global helper for fluent method chaining via `NumberCollection`.

```php
// Static usage
echo Number::pretify(1250000); // 1.2M

// Fluent usage
echo number(1250000)->pretify()->get(); // 1.2M
```

## Fluent Usage

The `number($value)` helper returns a `NumberCollection` instance, allowing you to chain multiple operations together. Call `get()` at the end of the chain to retrieve the final value.

```php
$result = number(123)
    ->toWords()
    ->get();
// "one hundred and twenty three"

$formatted = number(5)
    ->trailingZeros(3, 100)
    ->get();
// "005"
```

## Formatting & Presentation

### pretify / abbreviate

```php
static pretify(mixed $number, int $decimal_place = 0): string
static abbreviate(mixed $number, int $decimal_place = 0): string
```

Abbreviates large numbers into human-readable strings. `abbreviate` is an alias for `pretify`.

| Unit | Scale | Meaning     |
| :--- | :---- | :---------- |
| `K`  | 10³   | Thousand    |
| `M`  | 10⁶   | Million     |
| `B`  | 10⁹   | Billion     |
| `T`  | 10¹²  | Trillion    |
| `Q`  | 10¹⁵  | Quadrillion |

```php
Number::pretify(1500);    // "1.5K"
Number::pretify(1200000); // "1.2M"
```

### format

```php
static format(mixed $number, int $pos = 0): string
```

Formats a number with grouped thousands using standard formatting.

```php
Number::format(1250.5, 2); // "1,250.50"
```

### tosize

```php
static tosize(int $number): string
```

Converts a raw byte count into a human-readable file size format (b, kb, mb, gb, tb, pb).

```php
Number::tosize(2048); // "2 kb"
```

### trailingZeros

```php
static trailingZeros(int $number, int $pad, int $limit): string
```

Pads a number with leading zeros if it is below a specified limit.

```php
Number::trailingZeros(5, 3, 100); // "005"
Number::trailingZeros(150, 3, 100); // "150" (limit exceeded)
```

### ordinal

```php
static ordinal(int $num): string
```

Appends the correct English ordinal suffix (`st`, `nd`, `rd`, `th`) to a number.

```php
Number::ordinal(1);  // "1st"
Number::ordinal(22); // "22nd"
Number::ordinal(3);  // "3rd"
Number::ordinal(4);  // "4th"
```

## Conversions & Logic

### toWords

```php
static toWords(int $number, bool $default = false): string
```

Converts a numeric value into its English word equivalent. If `$default` is `true`, it uses the system's `NumberFormatter` (Intl extension required).

```php
Number::toWords(123); // "one hundred and twenty three"
```

### toRoman / fromRoman

```php
static toRoman(int $number): string
static fromRoman(string $roman): int
```

Converts between integers and standard Roman numerals.

```php
Number::toRoman(2024);   // "MMXXIV"
Number::fromRoman('XIV'); // 14
```

### toAlphabet

```php
static toAlphabet(int $number): string
```

Converts a number (1-26) to its corresponding uppercase letter (A-Z).

```php
Number::toAlphabet(1); // "A"
Number::toAlphabet(26); // "Z"
```

### clamp

```php
static clamp(int|float $number, int|float $min, int|float $max): int|float
```

Restricts a number to be within a specific range.

```php
Number::clamp(150, 0, 100); // 100
Number::clamp(-50, 0, 100); // 0
```

### percentage / toPercentage

```php
static percentage(int|float $value, int|float $total): float
static toPercentage(int|float $value, int $precision = 0): string
```

`percentage` calculates the raw percentage value (0-100). `toPercentage` rounds a value and appends the `%` symbol.

```php
Number::percentage(50, 200);   // 25.0
Number::toPercentage(25.45, 1); // "25.5%"
```

### random

```php
static random(int $min = 0, int|null $max = null): int
```

Generates a cryptographically secure random integer. If `$max` is null, it defaults to `PHP_INT_MAX`.

### toDecimal / toInteger

```php
static toDecimal(mixed $value): mixed
static toInteger(mixed $value): int
```

Scales values by 100. Useful for shifting between major units and minor units (like cents) for storage.

```php
Number::toInteger(10.50); // 1050
Number::toDecimal(1050);  // 10.5
```

## Related

- [Array Helper](array-helper.md) - Math operations on collections
- [Money Helper](money-helper.md) - Specialized currency handling
- [Format Helper](format-helper.md) - General data formatting
