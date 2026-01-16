# Dock Command Runner

A fluent, production-ready API for executing dock commands programmatically with comprehensive error handling and edge case coverage.

## Installation

The `DockCommand` class is available globally via the `dock()` helper function.

## Basic Usage

### Simple Command

```php
use Cli\Helpers\DockCommand;

// Using static make method
$result = DockCommand::make('migration:status')->run();

// Using global helper
$result = dock('migration:status')->run();

// Check result
if ($result->successful()) {
    echo $result->getOutput();
}
```

### With Arguments

```php
dock('package:uninstall')
    ->argument('Bridge')
    ->run();
```

### With Options

```php
dock('test')
    ->option('filter', 'UserTest')
    ->run();
```

### With Flags

```php
dock('package:uninstall')
    ->argument('Bridge')
    ->flag('system')
    ->flag('yes')
    ->run();
```

## Advanced Usage

### Complex Command Building

```php
$result = dock('package:install')
    ->argument('Tenancy')
    ->option('system')
    ->option('config', 'custom.php')
    ->flag('yes')
    ->flag('force')
    ->timeout(600)
    ->run();
```

### Multiple Arguments and Options

```php
dock('test')
    ->arguments(['UserTest', 'PostTest'])
    ->options(['filter' => 'UserTest', 'group' => 'unit'])
    ->flags(['coverage', 'verbose'])
    ->run();
```

### Timeout Configuration

```php
// Set timeout in seconds (default: 3600)
dock('queue:work')
    ->option('identifier', 'emails')
    ->timeout(300) // 5 minutes
    ->run();
```

### Retry Logic

```php
// Retry failed commands
$result = dock('migration:run')
    ->retry(3, 1000) // 3 retries, 1 second delay
    ->run();
```

### Environment Variables

```php
dock('test')
    ->env(['APP_ENV' => 'testing', 'DB_CONNECTION' => 'sqlite'])
    ->run();
```

### Dry Run Mode

```php
// Test command without executing
$result = dock('migration:run')
    ->option('file', '2024_01_01_create_user')
    ->dryRun()
    ->run();

echo $result->getOutput(); // php dock migration:run --file=2024_01_01_create_user
```

## Execution Modes

### Synchronous Execution

```php
// Run and wait for completion
$result = dock('test')->run();

if ($result->successful()) {
    echo "Tests passed!";
} else {
    echo "Tests failed: " . $result->getError();
}
```

### Quiet Execution

```php
// Suppress exceptions, always returns result
$result = dock('migration:run')->runQuietly();

if ($result->failed()) {
    logger()->error('Migration failed', [
        'error' => $result->getError(),
        'exit_code' => $result->getExitCode()
    ]);
}
```

### Asynchronous Execution

```php
// Non-blocking execution
$process = dock('queue:work')
    ->option('identifier', 'emails')
    ->runAsync();

// Do other work...

// Check if still running
if ($process->isRunning()) {
    echo "Queue worker is running...";
}

// Wait for completion
$process->wait();
```

## Result Handling

#### CommandResult Methods

```php
$result = dock('test')->run();

// Check success
$result->successful();  // bool
$result->failed();      // bool

// Get output
$result->getOutput();   // string
$result->getError();    // string

// Get metadata
$result->getExitCode();       // int
$result->getCommandLine();    // string
$result->getExecutionTime();  // float (seconds)

// Throw on failure
$result->throw(); // Throws RuntimeException if failed
```

### Fluent Error Handling

```php
dock('migration:run')
    ->run()
    ->onSuccess(function($result) {
        logger()->info('Migration successful', [
            'time' => $result->getExecutionTime()
        ]);
    })
    ->onFailure(function($result) {
        logger()->error('Migration failed', [
            'error' => $result->getError()
        ]);
    });
```

### Exception Handling

```php
try {
    dock('migration:run')
        ->option('file', 'invalid_migration')
        ->run()
        ->throw(); // Throws if failed

    echo "Migration successful!";
} catch (RuntimeException $e) {
    echo "Migration failed: " . $e->getMessage();
}
```

## Examples

### Package Installation

```php
function installPackage(string $package, bool $isSystem = false): bool
{
    $command = dock('package:install')
        ->argument($package)
        ->flag('yes');

    if ($isSystem) {
        $command->option('system');
    }

    $result = $command
        ->timeout(600)
        ->retry(2, 2000)
        ->run();

    if ($result->successful()) {
        logger()->info("Package {$package} installed successfully");
        return true;
    }

    logger()->error("Failed to install {$package}", [
        'error' => $result->getError()
    ]);

    return false;
}
```

### Running Tests with Coverage

```php
function runTestsWithCoverage(?string $filter = null): array
{
    $command = dock('test')
        ->flag('coverage')
        ->timeout(900);

    if ($filter) {
        $command->option('filter', $filter);
    }

    $result = $command->run();

    return [
        'success' => $result->successful(),
        'output' => $result->getOutput(),
        'time' => $result->getExecutionTime()
    ];
}
```

### Database Migration Runner

```php
function runMigrations(array $files = []): void
{
    $command = dock('migration:run');

    if (!empty($files)) {
        $command->option('file', implode(',', $files));
    }

    $result = $command
        ->timeout(300)
        ->retry(1, 5000)
        ->run();

    $result->throw(); // Throw exception if failed

    echo "Migrations completed in {$result->getExecutionTime()}s\n";
}
```

### Queue Worker Management

```php
function startQueueWorker(string $identifier, bool $daemon = false): void
{
    $command = dock('queue:work')
        ->option('identifier', $identifier);

    if ($daemon) {
        // Run asynchronously
        $process = $command->runAsync();

        logger()->info("Queue worker started", [
            'identifier' => $identifier,
            'pid' => $process->getPid()
        ]);
    } else {
        // Run synchronously
        $command->run();
    }
}
```

## Best Practices

1. **Always handle failures:**

   ```php
   $result = dock('command')->runQuietly();
   if ($result->failed()) {
       // Handle error
   }
   ```

2. **Use appropriate timeouts:**

   ```php
   dock('long-running-command')->timeout(1800)->run();
   ```

3. **Add retry logic for transient failures:**

   ```php
   dock('flaky-command')->retry(3, 2000)->run();
   ```

4. **Use dry-run for testing:**

   ```php
   if (app()->environment('testing')) {
       $result = dock('command')->dryRun()->run();
   }
   ```

5. **Log command execution:**
   ```php
   $result = dock('command')->run();
   logger()->info('Command executed', [
       'command' => $result->getCommandLine(),
       'time' => $result->getExecutionTime(),
       'success' => $result->successful()
   ]);
   ```

## Error Handling

The `DockCommand` class provides multiple layers of error handling:

1. **Validation errors** - Thrown immediately for invalid configurations
2. **Execution errors** - Captured in CommandResult
3. **Timeout errors** - Handled gracefully with proper error messages
4. **Retry logic** - Automatic retries for transient failures

```php
try {
    $result = dock('migration:run')
        ->timeout(300)
        ->retry(2)
        ->run();

    if ($result->failed()) {
        // Handle graceful failure
        logger()->error('Migration failed', [
            'error' => $result->getError(),
            'exit_code' => $result->getExitCode()
        ]);
    }
} catch (RuntimeException $e) {
    // Handle catastrophic failure
    logger()->critical('Command execution failed', [
        'exception' => $e->getMessage()
    ]);
}
```

## API Reference

### DockCommand Methods

| Method                                 | Description                      |
| -------------------------------------- | -------------------------------- |
| `make(string $command)`                | Create new instance with command |
| `command(string $command)`             | Set command name                 |
| `argument(string $value)`              | Add single argument              |
| `arguments(array $values)`             | Add multiple arguments           |
| `option(string $name, ?string $value)` | Add option                       |
| `options(array $options)`              | Add multiple options             |
| `flag(string $name)`                   | Add flag                         |
| `flags(array $flags)`                  | Add multiple flags               |
| `timeout(int $seconds)`                | Set timeout                      |
| `retry(int $times, int $delayMs)`      | Enable retry logic               |
| `env(array $env)`                      | Set environment variables        |
| `dryRun(bool $enabled)`                | Enable dry-run mode              |
| `buildCommand()`                       | Build command string             |
| `run()`                                | Execute synchronously            |
| `runQuietly()`                         | Execute without throwing         |
| `runAsync()`                           | Execute asynchronously           |

### CommandResult Methods

| Method                          | Description          |
| ------------------------------- | -------------------- |
| `successful()`                  | Check if successful  |
| `failed()`                      | Check if failed      |
| `getOutput()`                   | Get command output   |
| `getError()`                    | Get error output     |
| `getExitCode()`                 | Get exit code        |
| `getCommandLine()`              | Get executed command |
| `getExecutionTime()`            | Get execution time   |
| `throw()`                       | Throw if failed      |
| `onSuccess(callable $callback)` | Execute on success   |
| `onFailure(callable $callback)` | Execute on failure   |
