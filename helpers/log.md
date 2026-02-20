# Log Helper

The `Log` helper provides a clean, facade-like interface for writing to the application logs. It utilizes the underlying `FileLogger` to persist messages to storage.

## Basic Usage

```php
use Helpers\Log;

Log::info('User logged in', ['id' => 1]);
```

## Methods

### `channel`

Get a logger instance for a specific file. Using the `.log` extension is optional.

```php
Log::channel('verify')->info('OTP sent');
// Writes to App/storage/logs/verify.log
```

### `info`

Log an informational message.

```php
Log::info(string $message, array $context = []);
```

### `error`

Log an error message.

```php
Log::error(string $message, array $context = []);
```

### `warning`

Log a warning message.

```php
Log::warning(string $message, array $context = []);
```

### `critical`

Log a critical system error.

```php
Log::critical(string $message, array $context = []);
```

### `debug`

Log a debug message (useful for development).

```php
Log::debug(string $message, array $context = []);
```

## Context Data

All methods accept an optional `$context` array. This data is JSON-encoded and appended to the log entry, making it easy to review structured data.

```php
Log::error('Payment failed', [
    'order_id' => 'ord_123',
    'reason' => 'Insufficient funds',
    'gateway_response' => $response
]);
```

## Underlying Implementation

The `Log` facade proxies calls to `Helpers\File\FileLogger`. By default, logs are written to `App/storage/logs/anchor.log`.
