# Import

The Import package provides a bulk data import system that supports high-volume CSV processing with validation, error tracking, and real-time progress monitoring.

## Features

- **High-Volume Processing**: Efficiently handles massive files via chunked background processing.
- **Smart Mapping**: Declarative header mapping and data transformation.
- **Robust Validation**: Per-row validation with customizable error messages.
- **Error Tracking**: Detailed logs for failed rows, including specific column and value causing the issue.
- **Queue Integration**: Automatically offload processing to background workers.
- **Progress Monitoring**: Track successes, failures, and skipped rows in real-time.

## Installation

Import is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Import --packages
```

This will automatically:

- Run the migration for Import tables.
- Register the `ImportServiceProvider`.
- Publish the configuration file.

### Cloud Storage (S3/R2)

To use S3 or Cloudflare R2, you must configure the corresponding disk in your `App/Config/filesystems.php`.

```php
// App/Config/filesystems.php
'disks' => [
    's3' => [
        'driver' => 's3',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION'),
        'bucket' => env('AWS_BUCKET'),
        'endpoint' => env('AWS_ENDPOINT'), // Required for R2
    ],
],
```

The `Import` package will then transparently use these credentials when `->disk('s3')` is called.

### Configuration

Configuration file: `App/Config/import.php`

```php
return [
    'disk' => env('IMPORT_DISK', 'local'), // s3, r2, local
    'path' => 'imports',
    'chunk_size' => 500,
    'max_file_size' => 10485760, // 10MB
    'stop_on_error' => false,

    'queue' => [
        'enabled' => true,
        'connection' => 'default',
        'queue' => 'imports',
    ],
];
```

## Basic Usage

### Define an Importer

Implement the `Importable` contract to define your business logic:

```php
use Import\Contracts\Importable;
use App\Models\User;

class UsersImporter implements Importable
{
    public function model(): string
    {
        return User::class;
    }

    public function headers(): array
    {
        return ['name', 'email', 'role'];
    }

    public function rules(): array
    {
        return [
            'name' => [
                'required' => true,
                'type' => 'string',
                'maxlength' => 255
            ],
            'email' => [
                'type' => 'email',
                'strict' => 'disposable',
                'unique' => 'users.email'
            ],
            'role' => [
                'required' => false,
                'exist' => 'roles.slug'
            ],
        ];
    }

    public function messages(): array
    {
        return [
            'email.unique' => 'This email is already registered in our system.',
            'role.exist' => 'The selected role does not exist.',
        ];
    }

    public function map(array $row): array
    {
        return [
            'name' => $row['name'],
            'email' => strtolower(trim($row['email'])),
            'role' => $row['role'] ?? 'user',
        ];
    }

    public function handle(array $row): mixed
    {
        return User::updateOrCreate(['email' => $row['email']], $row);
    }
}
```

### Execute via Facade

```php
use Import\Import;

$history = Import::make(UsersImporter::class)
    ->file($uploadedPath)
    ->skipDuplicates(true)
    ->execute();

echo "Processed {$history->processed_rows} rows.";
```

## Use cases

#### Bulk Product Inventory Sync

Keep your product stock levels in sync with your warehouse by importing a daily CSV of over 50,000 SKUs.

#### Implementation

```php
use Import\Import;
use App\Importers\InventoryImporter;

public function syncInventory(Request $request)
{
    // High-volume file stored on S3
    $path = $request->file('csv')->store('imports', 's3');

    $history = Import::make(InventoryImporter::class)
        ->file($path)
        ->disk('s3')
        ->stopOnError(false)
        ->execute();

    return response()->json([
        'message' => 'Inventory sync initiated',
        'refid' => $history->refid
    ]);
}
```

#### Sample Data (JSON)

The `import_history` record state:

```json
{
  "refid": "imp_sync_998877",
  "status": "completed",
  "total_rows": 52400,
  "success_rows": 52385,
  "failed_rows": 15,
  "started_at": "2026-01-02 11:15:00",
  "completed_at": "2026-01-02 11:22:00"
}
```

## Package Integrations

### Audit Package (Traceability)

Every bulk operation is logged. You can track exactly who imported which file and when.

```php
use Audit\Audit;
use Import\Import;

public function processImport($path)
{
    $import = Import::make(UserImporter::class)
        ->file($path)
        ->execute();

    // The Import package already logs 'import.started' and 'import.completed'
    // but you can add custom business context:
    Audit::make()
        ->event('admin.bulk_user_invite')
        ->on($import)
        ->with('count', $import->total_rows)
        ->log();
}
```

### Event System (Progress)

Importing large files? Listen for progress events to update your UI.

```php
use Import\Events\RowProcessed;
use Helpers\Log;

// In your EventServiceProvider
public function handleRow(RowProcessed $event)
{
    $progress = ($event->currentRow / $event->totalRows) * 100;

    // Broadcast to user or log progress
    Log::info("Import progress: {$progress}%");
}
```

### Media Package

Import files are often managed via the `Media` package's underlying storage drivers, allowing for seamless transfers between `local` and `cloud` disks.

### Analytics & Reporting

Monitor system-wide import health via the `ImportAnalytics` service:

```php
$analytics = Import::analytics();

// 1. Core Health Metrics
$stats = $analytics->getStats();
/**
 * Sample Data:
 * {
 *   "total_imports": 450,
 *   "total_rows_processed": 125000,
 *   "average_success_rate": 94.5,
 *   "health_score": "A+"
 * }
 */

// 2. Common Validation Errors
$errors = $analytics->getCommonErrors();
/**
 * Sample Data:
 * [
 *   {"error": "The email has already been taken", "count": 120},
 *   {"error": "The selected role is invalid", "count": 45}
 * ]
 */

// Processing Trends
$trends = $analytics->getDailyTrends(30);
```

## Service API Reference

### Import (Facade)

| Method                | Description                                   |
| :-------------------- | :-------------------------------------------- |
| `make($importer)`     | Starts a fluent `ImportBuilder`.              |
| `history($userId)`    | Retrieves import history for a specific user. |
| `findByRefid($refid)` | Locates an import record by unique reference. |
| `analytics()`         | Returns the `ImportAnalytics` service.        |

### ImportBuilder (Fluent)

| Method                 | Description                                    |
| :--------------------- | :--------------------------------------------- |
| `file($path)`          | Sets the path to the CSV file.                 |
| `originalFilename($n)` | Sets the original name for the record.         |
| `skipDuplicates(bool)` | Whether to skip rows that already exist.       |
| `stopOnError(bool)`    | Whether to halt processing on the first error. |
| `execute()`            | Starts processing (or queues it).              |

### ImportHistory (Model)

| Attribute        | Type      | Description                               |
| :--------------- | :-------- | :---------------------------------------- |
| `status`         | `string`  | pending, processing, completed.           |
| `success_rows`   | `integer` | Number of records imported successfully.  |
| `failed_rows`    | `integer` | Number of records that failed validation. |
| `processed_rows` | `integer` | Total rows processed so far.              |

## Troubleshooting

| Error/Log          | Cause                                  | Solution                                |
| :----------------- | :------------------------------------- | :-------------------------------------- |
| "Invalid headers"  | CSV columns don't match `headers()`.   | Verify file structure against importer. |
| "File too large"   | Upload exceeds `max_file_size` config. | Increase limit or split the file.       |
| "Validation Error" | Row data violated `rules()`.           | Check `import_error` table for details. |

## Security Best Practices

- **File Validation**: Always verify that the uploaded file is a valid CSV/Excel file before passing it to the builder.
- **Queue Isolation**: Large imports should always run on a dedicated queue to prevent system-wide delays.
- **Sanitized Handlers**: The `handle()` method should use `updateOrCreate` or `insert` carefully to avoid unintentional data corruption.
