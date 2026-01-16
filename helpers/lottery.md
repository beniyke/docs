# Lottery Helper

The `Helpers\Lottery` class provides a controlled way to execute code based on probability (odds), weighted selection, and testing-friendly results.

> [!TIP]
> Use the `lottery()` global helper for quick access.

## Basic Usage

#### odds / check

```php
static odds(int $chances, int $outOf): Lottery
static check(int $chances, int $outOf): bool
```

`check` returns a boolean immediately. `odds` returns a fluent instance for more complex flows.

```php
// Simple boolean check (10% chance)
if (Lottery::check(1, 10)) {
    // 1 in 10 chance
}

// Fluent execution
Lottery::odds(1, 100)
    ->winner(fn() => "You won!")
    ->loser(fn() => "Try again")
    ->choose();
```

## Advanced Operations

#### choose

```php
choose(int $times = 1): mixed
```

Runs the lottery multiple times. If `$times > 1`, it returns an array of results.

#### pick (Weighted Selection)

```php
static pick(array $items): mixed
```

Selects one item from an associative array where the values represent **weights**.

```php
$reward = Lottery::pick([
    'Common Sword' => 70,    // 70% weight
    'Rare Shield'  => 25,    // 25% weight
    'Epic Armor'   => 5,     // 5% weight
]);
```

## Testing Hooks

The Lottery helper provides hooks to bypass randomness during testing, ensuring predictable results.

| Method            | Description                                                      |
| :---------------- | :--------------------------------------------------------------- |
| `alwaysWin()`     | All subsequent lottery checks will return `true`                 |
| `alwaysLose()`    | All subsequent lottery checks will return `false`                |
| `fix(array $seq)` | Force results to a specific sequence e.g., `[true, false, true]` |
| `normal()`        | Reset to standard random behavior                                |

```php
// In a unit test
Lottery::alwaysWin();
$result = actionThatUsesLottery();
// The lottery inside the action will always trigger the winner path
```

## Related

- [Number Helper](number-helper.md) - For generating random integers
- [Routine Helper](routine-helper.md) - Running probabilistic background tasks
