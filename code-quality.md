# Code Quality

Anchor provides a suite of tools to ensure your code remains clean, consistent, and error-free.

## Recommended Workflow

- **Development**: Run `php dock format` periodically to keep your code style consistent.
- **Review**: Run `php dock inspect` before committing to catch static analysis issues.
- **Deployment**: Run `php dock sail` as a final gatekeeper before merging or deploying.

## Inspecting Code (Analyze)

The `inspect` command performs static analysis and coding style checks to identify issues without modifying files.

```bash
php dock inspect
```

This command runs:

- **Pint** (in test mode) to check coding standards.
- **PHPStan** to perform static analysis on `App` and `System` directories.

If any issues are found, the command will exit with a failure status and report the errors.

## Formatting Code (Fix)

The `format` command automatically fixes coding style issues and removes unnecessary comments.

```bash
php dock format
```

### Options

| Option           | Description                                     |
| :--------------- | :---------------------------------------------- |
| `--dry-run`      | Preview changes without modifying files         |
| `--path`         | Specify target directory (default: App, System) |
| `--backup`       | Create `.bak` files before modifying            |
| `--max-size`     | Max file size in MB to process (default: 5)     |
| `--skip-pint`    | Skip running Pint after comment cleanup         |
| `--exclude`      | Directories to exclude (comma-separated)        |
| `--show-skipped` | Show all skipped files with reasons             |

### Aggressive Cleanup Logic

The `format` command uses an opinionated **Code Cleanup Engine** that goes beyond standard indentation fixes. It aims to reduce "visual noise" by removing redundant or obvious code artifacts.

#### 1. Redundant Docblock Removal

If a method has full PHP 8 type-hinting (parameters and return types), the formatter will remove docblocks that simply repeat that information.

- **Redundant**:
  ```php
  /**
   * @param string $name
   * @return void
   */
  public function setName(string $name): void
  ```
- **Preserved**: Docblocks containing `@throws`, complex generics (e.g., `array<string, User>`), or specific architectural explanations are always kept.

#### 2. Obvious Action Comments

The formatter identifies and removes comments that merely describe the next line of code without adding context.

- **Action**: `// Save user` followed by `$user->save()`.
- **Calculation**: `// Calculate total` followed by `$total = $price * $qty`.

#### 3. Narrative Heuristics

The cleanup engine is "narrative-aware." It understands the difference between a redundant comment and a helpful explanation.

- **Kept**: Comments containing keywords like `because`, `since`, `workaround`, `hack`, `important`, `warning`, or `note` are preserved as they provide valuable developer context.
- **Removed**: Simple "Step 1", "Step 2" markers or trivial "Create instance" comments.

### Configuration

You can configure the formatter in `.formatter.json` in the project root:

```json
{
  "paths": ["App", "System"],
  "backup": false,
  "maxFileSize": 5,
  "skipPint": false,
  "exclude": ["vendor", "node_modules", "storage"]
}
```

## Production Readiness (Verify)

The `sail` command is the ultimate assertion of readiness for production. It runs a comprehensive suite of checks in parallel.

```bash
php dock sail
```

### What it does

The command performs a "Pre-Flight Check" consisting of:

- **Inspection Check**: Runs `pint --test` (Style) and `format --check` (Comment Cleanup) in parallel.
- **Functionality Check**: Runs unit tests via `pest` (Correctness).

The command will only succeed if **ALL** checks pass. Use this command before pushing code or deploying to ensure your application is seaworthy.

```bash
Production Readiness Checks
===========================

1. Integrity & Style Inspection
Running Coding Style (Pint) and Comment Cleanup Check...

Coding Style (Pint) Passed.
Comment Cleanup Check Passed.

2. Operational Readiness & Testing
running Unit Tests...

Unit Tests Passed.

All checks passed! The application is ready for the Port (deployment). 🚀
VOYAGE CLEAR! All Inspections, Repairs, and Unit Tests passed. Vessel is cleared to set sail for the Port.
```
