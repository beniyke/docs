# Release

**Release** provides an automated, secure, way to manage application updates. It simplifies version checking, downloading, verifying, and applying updates, ensuring your application stays up-to-date with minimal downtime.

## Features

- **Automated Version Checking**: Regularly checks your configured endpoint for new releases.
- **Secure Downloads**: Downloads updates over secure channels.
- **Atomic Updates**: Uses a temporary staging area to extract and verify updates before applying them.
- **Backup & Restore**: Automatically backs up the application before updating, allowing for easy rollback in case of failure.
- **Maintenance Mode**: Automatically enables/disables maintenance mode during the update process to prevent inconsistencies.
- **Migration Scripts**: Supports pre-update (`beforeUpdate`) and post-update (`afterUpdate`) hook scripts for database migrations or other tasks.
- **Smart Exclusions**: Configurable exclusions to prevent overwriting specific user files or configurations.

## Installation

Release is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Release --packages
```

This will automatically:

- Publish the configuration.
- Register the `ReleaseServiceProvider`.

## Configuration

Configuration is located at `App/Config/release.php`. You can manage these settings via your `.env` file.

| Key                        | Type   | Default             | Description                                                                  |
| :------------------------- | :----- | :------------------ | :--------------------------------------------------------------------------- |
| `release.endpoint`         | string | `''`                | The URL endpoint to check for updates (e.g., `https://updates.example.com`). |
| `release.tmp_folder`       | string | `'App/storage/tmp'` | Temporary folder for downloading and extracting updates.                     |
| `release.maintenance_mode` | bool   | `true`              | Whether to enable maintenance mode during updates.                           |
| `release.backup`           | bool   | `true`              | Whether to backup the application before updating.                           |
| `release.authorized_users` | array  | `[]`                | List of user IDs authorized to perform updates (if applicable).              |
| `release.exclude`          | array  | `[]`                | List of files/directories to exclude from being overwritten during updates.  |

## Basic Usage

### Checking for Updates

You can manually check for updates using the CLI or programmatically via the `Release` facade.

**CLI:**

```bash
php dock release:check
```

**Code:**

```php
use Release\Release;

$update = Release::check();

if ($update) {
    // Update available: $update['version'], $update['archive']
    echo "New version available: " . $update['version'];
}
```

### Applying Updates

To apply the latest update found:

**CLI:**

```bash
php dock release:update
```

**Code:**

```php
use Release\Release;

if (Release::update()) {
    echo "Update successful!";
} else {
    echo "Update failed or no update available.";
}
```

## Advanced Usage

### The Update Workflow

When `Release::update()` is called, the `ReleaseManager` performs the following steps:

- **Check**: Queries the endpoint for `release.json` (containing version and archive URL).
- **Maintenance Mode**: Activates maintenance mode (if configured) to block user access.
- **Backup**: Creates a zip backup of the current application state (excluding storage/tmp).
- **Download**: Downloads the update archive to the temporary folder.
- **Extract**: Extracts the archive verified against checksums (if provided).
- **Pre-Update Hook**: Executing `upgrade.php` -> `beforeUpdate()` if it exists in the update package.
- **Overwrite**: Recursively copies new files to the application root, respecting `release.exclude`.
- **Post-Update Hook**: Executing `upgrade.php` -> `afterUpdate()` if it exists.
- **Cleanup**: Deletes temporary files and archives.
- **Restore**: Deactivates maintenance mode.

### Handling Failures

If any step fails, the `ReleaseManager` attempts to:

- **Restore**: Restores the application from the backup created in step 3.
- **Deactivate Maintenance Mode**: Deactivates maintenance mode.
- **Log Error**: Logs the error.
- **Throw Exception**: Throws an exception describing the failure.

## Analytics

The `Release` package provides an analytics service to track version history and backup health.

```php
use Release\Analytics;

// Get version statistics
$version = Analytics::getVersionStats();
// Returns: current_version, last_updated

// Get backup health and history
$backups = Analytics::getBackupStats();
// Returns: total_backups, total_size_bytes, last_backup, recent_backups
```

## Service API Reference

### Release (Facade)

| Method        | Description                                                                      |
| :------------ | :------------------------------------------------------------------------------- |
| `check()`     | Queries the endpoint and returns update metadata if available.                   |
| `update()`    | Triggers the full atomic update workflow (backup, download, extract, overwrite). |
| `analytics()` | Returns the `ReleaseAnalytics` service instance.                                 |

### Analytics (Facade)

| Method              | Description                                                 |
| :------------------ | :---------------------------------------------------------- |
| `getVersionStats()` | Returns current version and last update timestamp.          |
| `getBackupStats()`  | Returns total count, size, and history of system snapshots. |

### ReleaseManager

| Method                           | Description                                                      |
| :------------------------------- | :--------------------------------------------------------------- |
| `download(string $file)`         | Securely retrieves update archives using the robust HTTP client. |
| `copyWithExclusions($src, $dst)` | Safely synchronizes new files while protecting user data.        |

### BackupService (Release)

| Method                    | Description                                                     |
| :------------------------ | :-------------------------------------------------------------- |
| `create(string $version)` | Creates a pre-update system snapshot for rollback safety.       |
| `restore(string $path)`   | Reverts the application to a previous state in case of failure. |

## Best Practices

- **Backup Configuration**: Ensure your `release.backup` setting is enabled in production environments. This is your safety net.
- **Exclusions**: Carefully configure `release.exclude` to protect user-generated content (like `storage/uploads`) or custom configuration files from being overwritten.
- **Environment Variables**: Use `.env` to secure your update endpoint URL if it contains sensitive tokens.
