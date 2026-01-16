# Watcher

Watcher is a powerful, low-overhead monitoring and debugging tool integrated directly into the framework. It records requests, database queries, exceptions, jobs, and file logs, providing valuable insights into your application's performance and health.

## Installation

Watcher is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Watcher --packages
```

This will automatically:

- Publish the configuration to `App/Config/watcher.php`
- Register the `WatcherServiceProvider`
- Inject `WatcherMiddleware` into the `web` middleware group
- Run database migrations for the `watcher_entries` table

### Verify Database

Ensure the monitoring table is ready:

```bash
php dock migration:status
```

## Features

- **Request Monitoring**: Tracks HTTP requests, response times, status codes, and IPs.
- **SQL Query Insights**: Hooks into the database connection to log queries and their durations.
- **Exception Tracking**: Automatically captures unhandled exceptions with full context.
- **Job Monitoring**: Monitors queue worker events to track job success and duration.
- **File Log Interception**: Optionally captures standard file logs into the database for centralized searching.
- **User Impersonation Tracking**: Automatically listens for `Ghost` package events to log administrative impersonation activities.
- **Threshold Alerts**: Built-in manager to trigger notifications (Email, Slack, Webhook) when performance degrades.
- **Intelligent Batching**: Buffers entries in memory and flushes them at the end of the request to minimize I/O overhead.

## Configuration

### Sampling & Performance

Control the volume of data stored by adjusting sampling rates (0.0 to 1.0):

```php
'sampling' => [
    'request' => 1.0,    // 100%
    'query' => 1.0,      // 100%
    'exception' => 1.0,  // Always record errors
    'log' => 0.1,        // Only 10% of standard logs
],
```

### Batching Writes

Watcher uses a `BatchRecorder` to group database writes:

```php
'batch' => [
    'enabled' => true,
    'size' => 50,           // Flush after 50 entries
    'flush_interval' => 5,  // Or after 5 seconds
],
```

### Sensitive Data Redaction

Ensure passwords and tokens never hit your monitoring tables:

```php
'filters' => [
    'redact_fields' => ['password', 'token', 'secret', 'api_key', 'credit_card'],
],
```

## Usage

### Recording Events Manually

While Watcher is automatic, you can record custom events via the `Watcher` facade:

```php
use Watcher\Watcher;

Watcher::record('user_action', ['action' => 'login', 'user_id' => 123]);
```

### Accessing Analytics

Use the `analytics()` method to get performance metrics:

```php
use Watcher\Watcher;

// Get summary of the last 24 hours
$perf = Watcher::analytics()->getPerformanceMetrics('24h');

// Get specific statistics
$slowQueries = Watcher::analytics()->getSlowQueries(limit: 10);
```

## Alerts

Watcher includes an `AlertManager` accessible via the facade.

### Running Alert Checks

You can trigger alert checks manually or via a scheduled task:

```php
use Watcher\Watcher;

Watcher::alerts()->checkThresholds();
```

## Data Retention

To keep your database lean, Watcher includes a retention policy.

```php
'retention' => [
    'request' => 7,      // 7 days
    'query' => 7,        // 7 days
    'exception' => 30,   // 30 days
],
```

Run the cleanup command periodically:

```bash
php dock watcher:cleanup
```
