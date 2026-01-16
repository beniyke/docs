# String Helper

The `Helpers\String` namespace provides a robust suite for string manipulation, inflections, and UUID generation.

> Use the global helpers `str()`, `text()`, `plural()`, and `inflect()` for convenient access.

## Str

The `Helpers\String\Str` class is the primary utility for common string operations.

#### random / password

```php
static random(string $type = 'alnum', int $len = 32): string
static password(int $length = 32, array $options = []): string
static refid(): string
```

- `random`: Generates a CSPRNG string. Types: `alnum`, `alpha`, `numeric`, `hexdec`, `distinct`, `secure`.
- `password`: Generates a strong password. Options: `letters`, `numbers`, `symbols`, `spaces`.
- `refid`: Generates a unique 16-character alphanumeric reference ID (shorthand for `random('alnum', 16)`).
- `length`: Returns the length of the string (multi-byte safe).
- `repeat`: Repeats a string $n$ times.
- `reverse`: Reverses the string.

#### Security & Escaping

```php
static htmlEscape(mixed $var, bool $doubleEncode = true): mixed
static htmlentityEncode(string $str): string
static htmlentityDecode(string $str): string
```

Essential for preventing XSS. `htmlEscape` recursively handles arrays.

#### Entities

```php
static htmlcharacterEncode(string $str): string
static htmlcharacterDecode(string $str): string
static htmlentityEncode(string $str): string
static htmlentityDecode(string $str): string
static htmlencodeNewline(?string $string, string $connector = '&#10;'): string
```

#### Truncation & Padding

```php
static limit(string $string, int $limit = 100, string $end = '...'): string
static limitWords(string $string, int $limit = 100, string $end = '...'): string
static words(string $string, int $words = 100, string $end = '...'): string (Alias for limitWords)
static padLeft(string $string, int $length, string $padString = ' '): string
static padRight(string $string, int $length, string $padString = ' '): string
static padBoth(string $string, int $length, string $pad = ' '): string
```

#### Border Control

```php
static start(string $string, string $prefix): string
static finish(string $string, string $suffix): string
```

Ensures a string begins or ends with a specific substring without duplicating it.

#### Casing & Transformation

```php
static slug(string $string, string $replacement = '-'): string
static removeSlug(string $string, string $replacement = '-'): string
static transliterate(string $string): string
static title(string $string): string
static lower(string $string): string
static upper(string $string): string
static camel(string $string): string
static snake(string $string): string
static kebab(string $string): string
static studly(string $string): string (PascalCase)
static ucfirst(string $string): string
```

`transliterate` converts non-ASCII characters to their closest ASCII equivalents.

#### Search & Inspection

```php
static contains(string $haystack, string|array $needles, bool $ignoreCase = false): bool
static startsWith(string $haystack, string $needle, bool $ignoreCase = false): bool
static endsWith(string $haystack, string $needle, bool $ignoreCase = false): bool
static is(string $pattern, string $value): bool (Wildcard match e.g. `admin/*`)
static isAlphaNumeric(string $string): bool
static isLowercase(string $string): bool
static isUppercase(string $string): bool
```

#### Substrings & Extraction

```php
static before(string $subject, string $search): string
static after(string $subject, string $search): string
static between(string $subject, string $from, string $to): string
static substr(string $string, int $start, ?int $length = null): string
```

#### Masking

```php
static mask(string $string, string $character = '*', int $index = 0, ?int $length = null): string
static maskEmail(string $email): string
static maskSensitiveData(array $data, array $keywords, string $mask = '*'): array
```

#### Utilities

```php
static prettyImplode(array $items, string $conjunction = 'and'): string
static makeUrlClickable(string $string): string
static touch(array $array, array $actions): array
static replace(string $str, string|array $find, string|array $replace): string
static replaceArray(string $search, array $replacements, string $subject): string
static swap(array $map, string $subject): string
static convertEncoding(string $string, string $toCharset, string $fromCharset = 'UTF-8'): string
static shortenWithInitials(string $string): string
```

- `prettyImplode`: Creates a list like "Apples, Oranges, and Bananas".
- `makeUrlClickable`: Wraps URLs in `<a>` tags.
- `touch`: Applies `Str` methods to array values via a map.
- `swap`: Performs multiple replacements via an associative array.

## Text

Extends `Inflector` with specialized text processing for larger content.

#### wrap

```php
static wrap(string $str, int $charlim = 76): string
```

Wraps text to a limit. Supports `{unwrap}...{/unwrap}` tags to exempt specific blocks.

#### censor

```php
static censor(string $str, array $censored, string $replacement = ''): string
```

Redacts words. Supports `*` as a wildcard (e.g., `f*ck`).

#### trim

```php
static trim(string $input, int $length, bool $ellipses = true, bool $strip_html = true): string
```

Unlike `Str::limit`, this trims to the last full space before the limit.

#### estimated_read_time

```php
static estimated_read_time(string $content, int $wpm = 200, bool $seconds = false): string
```

## Inflector

Handles English pluralization rules.

#### pluralize / singularize / inflect

```php
static pluralize(string $word): string
static singularize(string $word): string
static inflect(string $word, int $count): string
```

`inflect` automatically pluralizes the word if `$count > 1`.

## Uuid Generator

Generates RFC-compliant Unique IDs.

#### Generation

| Method            | Type          | Best For                               |
| :---------------- | :------------ | :------------------------------------- |
| `v1()`            | Time-based    | Distributed systems tracking           |
| `v4()`            | Random        | General unique IDs                     |
| `v7()`            | Time-ordered  | **Database Primary Keys** (sequential) |
| `v8(data, ns)`    | Custom        | Name-based/Hashed IDs                  |
| `nameBased(name)` | Deterministic | Hashing a string into a UUID           |

#### Inspection & Constants

- `isValid(string $uuid): bool`
- `getVersion(string $uuid): ?int`
- `isNil(string $uuid): bool`
- `UuidGenerator::NIL` / `UuidGenerator::MAX`

## Related

- [Number Helper](number-helper.md) - Numeric conversions
- [Data Helper](data-helper.md) - Handling numeric/string datasets
- [Validation Helper](validation-helper.md) - String format validation
- [HTML Helper](html-helper.md) - Generating HTML from strings
