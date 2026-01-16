# DateTime Helper

The `Helpers\DateTimeHelper` class extends **Carbon** to provide application-specific logic for handling dates, timezones, and ranges.

> Use the `datetime()` global helper for quick access to `DateTimeHelper`.

## Instance Creation

#### now / immutable / mutable

```php
static now($tz = null): static
static immutable($date = null, $tz = null): static
static mutable($date = null, $tz = null): Carbon
```

Returns instances of date objects. `now` and `immutable` return `DateTimeHelper` (immutable), while `mutable` returns a standard `Carbon` instance.

#### createFrom

```php
static createFrom(string $date): static
```

Alias for `parse()`. Creates a `DateTimeHelper` instance from a string.

## Core Logic

#### safeParse

```php
static safeParse(?string $date): ?Carbon
```

Safely parses a date string. Returns a Carbon instance or `null` if the input is empty or invalid.

#### convert / toUtc

```php
static convert(string $date, string $from_tz, string $to_tz): string
static toUtc(string $date, string $from_tz): string
```

Converts a date between timezones. `toUtc` is a shortcut for converting UI inputs to database storage format.

#### getTimezones

```php
static getTimezones(): array
```

Returns a flat array of all PHP-supported timezone identifiers (e.g., `Africa/Lagos`).

## Ranges & Intervals

#### prepareDate

```php
static prepareDate(string $date): array
```

Parses a date string into an array `[start, end]`. It supports range strings (e.g., "2023-01-01 to 2023-01-31").

#### Boundary Helpers

All boundary helpers return an array `[start_datetime, end_datetime]` in database format (`Y-m-d H:i:s`).

| Method                       | Description                                           |
| :--------------------------- | :---------------------------------------------------- |
| `startAndEndOfDay(date)`     | From 00:00:00 to 23:59:59                             |
| `startAndEndOfWeek(date)`    | Start and end of the week for the given date          |
| `startAndEndOfThisMonth()`   | From 1st 00:00:00 to last day 23:59:59                |
| `startAndNowOfThisMonth()`   | From 1st 00:00:00 to current time                     |
| `startAndEndOfQuarter(q, y)` | Start and end of the specified quarter (1-4) and year |
| `startAndEndOfYear(year)`    | From Jan 1st to Dec 31st                              |

## Formatting & Interpretation

#### formatShortDate / formatFriendlyDatetime

```php
static formatShortDate(string $date): string
static formatFriendlyDatetime(string $date): string
```

Returns pre-configured formats:

- `formatShortDate`: `Wed, Oct 31 2025`
- `formatFriendlyDatetime`: `Oct 31, 2025 at 05:14 PM`

#### timeAgo / diffForHumansShort

```php
static timeAgo(string $date, bool $with_tense = true): string
static diffForHumansShort(string $date): string
```

- `timeAgo`: Returns full localized string (e.g., "3 hours ago", "just now").
- `diffForHumansShort`: Returns compact string (e.g., "3h", "5m", "1d").

#### interpreteDate

```php
static interpreteDate(?string $date, bool $dateonly = true): string
```

Converts a date (or range) into a human-readable summary (e.g., "Jan 1, 2023 - Jan 31, 2023").

## Checks & Math

#### Calendar Checks

- `isDateToday(string $date): bool`
- `isDateYesterday(string $date): bool`
- `isDateTomorrow(string $date): bool`
- `isDateWeekend(string $date): bool`
- `isDateBusinessDay(string $date): bool`
- `isDateBirthday(string $date): bool` (Checks month/day match)
- `isHoliday(string $date, array $holidayDates): bool`

#### Future/Past Checks

- `checkIfFuture(string $date): bool`
- `checkIfPast(string $date): bool`

#### getAge

```php
static getAge(string $birthday): int
```

Calculates age in years based on a birthday string.

## Related

- [Number Helper](number-helper.md) - For numeric date IDs
- [Data Helper](data-helper.md) - Formatting datasets containing dates
- [Validation Helper](validation-helper.md) - Validating date input formats
