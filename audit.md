# Audit

The Audit package provides a tamper-proof activity logging and audit trail system. It tracks model changes, user actions, and system events with built-in integrity verification.

## Features

- **Automated Tracking**: Automatically log model creation, updates, and deletions via traits.
- **Tamper Detection**: Secure SHA-256 HMAC checksums for every log entry to prevent database manipulation.
- **Granular Control**: Easily exclude sensitive attributes like passwords or tokens from logs.
- **Contextual Metadata**: Automatically captures user IDs, IP addresses, and user-agent strings.
- **Retention Management**: Built-in cleanup tools to manage log growth based on configurable periods.
- **Advanced Querying**: Fluent API for retrieving changes and history for any model or user.

## Installation

Audit is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Audit --packages
```

This will automatically:

- Run the migration for Audit tables.
- Register the `AuditServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/audit.php`

```php
return [
    'enabled' => env('AUDIT_ENABLED', true),
    'events' => [
        'created' => true,
        'updated' => true,
        'deleted' => true,
        'restored' => true,
    ],
    'retention_days' => 90,
    'checksum' => [
        'enabled' => true,
        'algorithm' => 'sha256',
    ],
    'excluded_attributes' => [
        'password', 'remember_token', 'api_token', 'two_factor_secret',
    ],
    'queue' => [
        'enabled' => env('AUDIT_QUEUE_ENABLED', false),
        'connection' => env('AUDIT_QUEUE_CONNECTION', 'default'),
        'queue' => env('AUDIT_QUEUE_NAME', 'audit'),
    ],
];
```

## Basic Usage

### Enable Model Auditing

Add the `HasAuditTrail` trait to any model:

```php
use Audit\Traits\HasAuditTrail;

class Project extends BaseModel
{
    use HasAuditTrail;

    // Optional: Exclude specific fields
    protected array $auditExclude = ['internal_notes'];
}
```

### Manual Event Logging

Use the fluent `LogBuilder` for descriptive manual entries:

```php
use Audit\Audit;

// Fluent, expressive logging
Audit::make()
    ->event('payment.verified')
    ->metadata(['amount' => 500, 'gateway' => 'stripe'])
    ->with('processor_id', 'ch_12345')
    ->log();

// Log against a specific resource (Auditable)
Audit::make()
    ->event('document.signed')
    ->on($document)
    ->by($user)
    ->log();
```

## Advanced Features

### Integrity Verification

Ensure logs haven't been modified externally:

```php
$log = Audit::history()->find($id);

if (!Audit::verify($log)) {
    throw new SecurityException("Audit log integrity compromised!");
}
```

### Global Analytics

Monitor system activity and security health:

```php
$analytics = Audit::analytics();

// Get event distribution (created vs updated vs deleted)
$stats = $analytics->getEventStats();

// Identify high-activity users
$activeUsers = $analytics->getTopUsers(5);
```

## Use Cases

### High-Security Document Signing

In a legal application, every time a document is signed, we need a tamper-proof record of the user, their IP, and the document version.

#### Implementation

```php
use Helpers\DateTimeHelper;

public function signDocument(Document $doc, User $user)
{
    $doc->update([
        'status' => 'signed',
        'signed_at' => DateTimeHelper::now()
    ]);

    Audit::make()
        ->event('legal.document_signature')
        ->on($doc)
        ->by($user)
        ->metadata([
            'version' => $doc->version,
            'agreement_type' => 'NDA',
            'hash_verified' => true
        ])
        ->log();
}
```

#### Sample Data (JSON Representation)

What is actually stored in the `audit_log` table:

```json
{
  "refid": "aud_secure_987654321",
  "event": "legal.document_signature",
  "user_id": 42,
  "user_ip": "192.168.1.105",
  "auditable_type": "App\\Models\\Document",
  "auditable_id": 1024,
  "old_values": { "status": "pending" },
  "new_values": { "status": "signed" },
  "metadata": {
    "version": "v2.1",
    "agreement_type": "NDA",
    "hash_verified": true
  },
  "checksum": "8f3e2... (HMAC-SHA256)",
  "created_at": "2026-01-02 10:25:00"
}
```

## Service API Reference

### Audit (Facade)

| Method               | Description                                     |
| :------------------- | :---------------------------------------------- |
| `make()`             | Start a fluent `LogBuilder`.                    |
| `log($event, $data)` | records a custom audit entry (Legacy/Simple).   |
| `verify($log)`       | Validates the checksum integrity of a log.      |
| `history()`          | Returns a query builder for all audit logs.     |
| `cleanup($days)`     | Purges logs older than the specified retention. |
| `analytics()`        | Returns the `AuditAnalytics` service.           |

### HasAuditTrail (Trait)

| Method              | Description                                    |
| :------------------ | :--------------------------------------------- |
| `auditLogs()`       | Relationship returning all logs for the model. |
| `lastAuditLog()`    | Returns the most recent audit entry.           |
| `getAuditChanges()` | Compares old/new values from the last update.  |

### AuditLog (Model)

| Attribute    | Type     | Description                            |
| :----------- | :------- | :------------------------------------- |
| `event`      | `string` | created, updated, deleted, or custom.  |
| `old_values` | `array`  | State of attributes before the change. |
| `new_values` | `array`  | State of attributes after the change.  |
| `user_ip`    | `string` | Originating IP address of the action.  |
 
## Automation
 
The Audit package includes an automated cleanup task that is registered in the framework scheduler. This ensures logs are purged based on your retention configuration.
 
```php
// packages/Audit/Schedules/AuditCleanupSchedule.php
namespace Audit\Schedules;
 
use Cron\Interfaces\Schedulable;
use Cron\Schedule;
 
class AuditCleanupSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('audit:cleanup')
            ->weekly();
    }
}
```

## Troubleshooting

| Error/Log                | Cause                                   | Solution                                    |
| :----------------------- | :-------------------------------------- | :------------------------------------------ |
| "Integrity Check Failed" | Log entry data modified in DB directly. | Investigate potential DB breach.            |
| "Missing old_values"     | Initial 'created' event has no history. | Normal behavior for first entry.            |
| "High Disk Usage"        | Large volume of logs not being purged.  | Run `php dock audit:cleanup` or adjust TTL. |

## Security Best Practices

- **Restrict Exclusions**: Ensure sensitive PII and credentials are explicitly added to `auditExclude`.
- **External Storage**: For high-security environments, periodically export and sign logs to an external immutable storage.
- **Proactive Monitoring**: Use `Audit::verify()` in sensitive admin areas to alert on any tampering.
