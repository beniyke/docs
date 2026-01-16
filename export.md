# Export

The Export package provides a data export and reporting system that supports multiple formats, queued background processing, and comprehensive history tracking.

## Features

- **Multi-Format Support**: Native support for CSV and JSON with extensible architecture.
- **Queued Processing**: Offload massive data exports to background workers.
- **Fluent Builder**: Chainable API for configuring filenames, disks, and formats.
- **History Tracking**: Complete audit trail of every export with status monitoring.
- **Automated Cleanup**: Configurable retention periods for generated files.
- **Analytics**: Real-time reporting on export volume, success rates, and file usage.

## Installation

Export is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Export --packages
```

This will automatically:

- Run database migrations for `export` table.
- Register the `ExportServiceProvider`.
- Publish the configuration file.

### Cloud Storage (S3/R2)

Exports can be stored on cloud disks. Configure your `s3` or `r2` disk in `App/Config/filesystems.php`:

```php
// App/Config/filesystems.php
'disks' => [
    's3' => [
        'driver' => 's3',
        'key' => env('AWS_ACCESS_KEY_ID'),
        // ... secret, region, bucket ...
    ],
],
```

### Configuration

Configuration file: `App/Config/export.php`

```php
return [
    'disk' => 'local',
    'path' => 'exports',
    'chunk_size' => 1000,
    'retention_days' => 7,

    'queue' => [
        'enabled' => true,
        'connection' => 'default',
        'queue' => 'exports',
    ],
];
```

## Basic Usage

### Define an Exporter

Implement the `Exportable` contract to define your data source:

```php
use Export\Contracts\Exportable;

class UsersExporter implements Exportable
{
    public function query(): mixed
    {
        return User::query();
    }

    public function headers(): array
    {
        return ['ID', 'Name', 'Email'];
    }

    public function map(mixed $row): array
    {
        return [$row->id, $row->name, $row->email];
    }

    public function filename(): string
    {
        return 'users_export';
    }
}
```

### Execute via Facade

Use the `Export` facade for fluent orchestration:

```php
use Export\Export;

// Background Export (Queued)
$history = Export::make(UsersExporter::class)
    ->csv()
    ->filename('users_august')
    ->execute();

// Direct Download (In-browser)
$result = Export::make(new UsersExporter())->json()->download();

// Optimized framework response download
return response()->download(
    $result['content'],
    $result['filename'],
    $result['mime_type']
);
```

## Use Cases

#### Monthly Financial Report

Generate a high-volume CSV of all paid invoices for the previous month, accessible via a background queue.

#### Implementation

```php
use Export\Export;
use Helpers\DateTimeHelper;

public function generateMonthlyReport()
{
    $lastMonth = DateTimeHelper::now()->subMonth()->format('F Y');

    // Dispatch to background processing
    $history = Export::make(InvoiceExporter::class)
        ->csv()
        ->filename("Financial_Report_{$lastMonth}")
        ->queue()
        ->execute();

    return response()->json([
        'message' => 'Report is being generated',
        'refid' => $history->refid
    ]);
}
```

#### Sample Data (JSON)

The `export_history` record state:

```json
{
  "refid": "exp_secure_456789",
  "status": "completed",
  "exporter_class": "App\\Exporters\\InvoiceExporter",
  "format": "csv",
  "rows_count": 15420,
  "file_size": 2450000,
  "path": "exports/Financial_Report_December_2025.csv",
  "completed_at": "2026-01-02 11:10:00"
}
```

## Package Integrations

### Money Package (Financial Audits)

The `Export` package is the native way to generate bulk financial reports.

```php
use Export\Export;
use Money\Models\Transaction;

// End-to-End: Exporting all high-value transactions for the current year
public function exportHighValueTransactions()
{
    $query = Transaction::where('amount', '>', 1000)
        ->whereYear('created_at', date('Y'));

    return Export::make(new TransactionExporter($query))
        ->csv()
        ->filename("High_Value_Transactions_" . date('Y'))
        ->download();
}
```

### Audit Package (Traceability)

Every export is logged. For sensitive data, you can add extra context:

```php
use Audit\Audit;
use Export\Export;

public function exportPayroll($users)
{
    $history = Export::make(new PayrollExporter($users))
        ->queue()
        ->execute();

    // The Export package logs 'export.started' automatically.
    // We add a security audit entry for sensitive payroll access:
    Audit::make()
        ->event('security.payroll_export_initiated')
        ->on($history)
        ->with('affected_users', $users->count())
        ->log();
}
```

### Analytics & Reporting

Monitor system-wide export activity via the `ExportAnalytics` service:

```php
$analytics = Export::analytics();

// 1. Core Success Metrics
$stats = $analytics->getStats();
/**
 * Sample Data:
 * {
 *   "total_exports": 850,
 *   "success_rate": 98.5,
 *   "failed_exports": 12,
 *   "avg_processing_time": "14s"
 * }
 */

// 2. Format Popularity
$formats = $analytics->getByFormat();
/**
 * Sample Data:
 * {
 *   "csv": 620,
 *   "json": 210,
 *   "pdf": 20
 * }
 */

// 3. Peak Export Times (Daily Trends)
$trends = $analytics->getDailyTrends(7);
/**
 * Sample Data:
 * [
 *   {"date": "2026-01-01", "completed": 45, "failed": 0},
 *   {"date": "2026-01-02", "completed": 52, "failed": 1}
 * ]
 */
```

```

## Service API Reference

### Export (Facade)

| Method                   | Description                                        |
| :----------------------- | :------------------------------------------------- |
| `make($exporter)`        | Starts a fluent `ExportBuilder`.                   |
| `queue($exporter, $opt)` | Directly queues an export with options.            |
| `history($userId)`       | Retrieves export history for a specific user.      |
| `findByRefid($refid)`    | Locates an export record by unique reference.      |
| `download($export)`      | Prepares download metadata for a completed record. |
| `cleanup($days)`         | Purges old export records and files.               |
| `analytics()`            | Returns the `ExportAnalytics` service.             |

### ExportBuilder (Fluent)

| Method            | Description                                            |
| :---------------- | :----------------------------------------------------- |
| `format($format)` | Sets output format (CSV, JSON, etc).                   |
| `csv() / json()`  | Quick shorthand for specific formats.                  |
| `filename($name)` | Sets a custom name for the output file.                |
| `disk($disk)`     | Specifies the storage disk to use.                     |
| `queue()`         | Signals that processing should be backgrounded.        |
| `execute()`       | Returns an `ExportHistory` model.                      |
| `download()`      | Returns raw file content for immediate browser action. |

### ExportHistory (Model)

| Attribute    | Type      | Description                             |
| :----------- | :-------- | :-------------------------------------- |
| `status`     | `string`  | pending, processing, completed, failed. |
| `rows_count` | `integer` | Number of records exported.             |
| `file_size`  | `integer` | Size of the generated file in bytes.    |
| `refid`      | `string`  | Unique traceable reference ID.          |

## Console Commands

| Command          | Description                             |
| :--------------- | :-------------------------------------- |
| `export:cleanup` | Automated task for purging old files.   |
| `export:stats`   | Visualize current system export volume. |
| `export:retry`   | Re-queue failed export attempts.        |

## Troubleshooting

| Error/Log                  | Cause                                           | Solution                                   |
| :------------------------- | :---------------------------------------------- | :----------------------------------------- |
| "Disk space full"          | Storage disk has reached its quota.             | Cleanup old exports or expand storage.     |
| "Queue not processing"     | Worker is not running for 'exports'.            | Run `php dock queue:work --queue=exports`. |
| "Exporter class not found" | Provided class name is incorrect or misspelled. | Verify the FQCN in the `make()` call.      |
| "Format not supported"     | Attempting to use unconfigured format.          | Check `App/Config/export.php`.             |

## Security Best Practices

- **Access Control**: Always check if the user has permission to view the data before initiating an export.
- **Data Sanitization**: Exporters should only query and map fields that the user is authorized to see.
- **Retention Policy**: Regularly run `export:cleanup` to ensure sensitive data doesn't sit on the disk indefinitely.
- **Queue Isolation**: Use a dedicated queue for exports to prevent large reports from blocking critical system notifications.
```
