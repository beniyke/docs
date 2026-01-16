# VarDump Helper

The `Helpers\VarDump` class provides enhanced variable inspection and debugging output, supporting both browser and CLI environments with automatic formatting and syntax coloring.

> Use the `dump()` and `dd()` global helpers for quick access.

## Features

- **Automatic Environment Detection**: Formats output as HTML in browsers and colored ANSI text in CLI.
- **Recursion Detection**: Prevents infinite loops when dumping circular references.
- **Rich Object Inspection**: Shows visibility (public/protected/private) and property types.
- **Interactive Browsing**: In-browser dumps include toggleable sections for large arrays and objects.

## Methods

#### dump

```php
dump(mixed ...$vars): void
```

Evaluates and outputs one or more variables in a human-readable format. The script execution continues after the output.

- **Use Case**: Inspecting data mid-flow without stopping the application (e.g., checking values during a loop).

#### dd

```php
dd(mixed ...$vars): void
```

"Dump and Die". Output variables in a readable format and immediately terminate the script.

- **Use Case**: Stopping execution to inspect the exact state of variables at a specific point in the code.

## Customization

The class uses a predefined color palette for different data types. In the browser, it injects custom CSS and JS for the interactive experience.

```php
use Helpers\VarDump;

(new VarDump())->dump($complexArray, $userObject);
```

## Global Helpers

#### dump()

`dump(mixed ...$vars): void`

Shortcut for `(new VarDump())->dump()`.

#### dd()

`dd(mixed ...$vars): void`

Shortcut for `(new VarDump())->dd()`.

## Related

- [Benchmark](benchmark-helper.md) - Pairing variable dumps with performance data
- [Log](log-helper.md) - For inspection that shouldn't disrupt the output
