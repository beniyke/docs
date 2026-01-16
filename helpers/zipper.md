# Zipper Helper

The `Helpers\File\Zipper\Zipper` class provides robust ZIP archive creation and extraction with support for password protection (AES-256) and multiple file/directory handling.

```php
// Create a ZIP archive
$result = (new Zipper())
    ->add(['/path/to/file1.txt', '/path/to/file2.pdf'])
    ->path('/path/to/output')
    ->save('archive.zip');

// Extract an archive
$result = (new Zipper())
    ->file('/path/to/archive.zip')
    ->path('/path/to/extract')
    ->extract();
```

## Creating Archives

### add

```php
add(array $files): self
```

Adds files to be archived. Accepts an array of absolute file paths.

```php
$zipper = (new Zipper())->add([
    '/path/to/document.pdf',
    '/path/to/image.jpg',
    '/path/to/report.xlsx'
]);
```

### path

```php
path(string $path): self
```

Sets the destination directory for the output archive.

```php
$zipper->path('/path/to/output/directory');
```

### save

```php
save(string $name): Result
```

Creates the ZIP archive with the specified filename. Returns a `Result` object.

```php
$result = (new Zipper())
    ->add(['/path/to/files/...'])
    ->path('/path/to/output')
    ->save('backup.zip');

if ($result->isSuccess()) {
    echo "Created: " . $result->getData('zip_path');
} else {
    echo "Error: " . $result->getMessage();
}
```

## Extracting Archives

### file

```php
file(string $file): self
```

Sets the ZIP file to extract.

```php
$zipper->file('/path/to/archive.zip');
```

### extract

```php
extract(): Result
```

Extracts the archive to the path set via `path()`. Returns a `Result` object.

```php
$result = (new Zipper())
    ->file('/path/to/archive.zip')
    ->path('/path/to/destination')
    ->extract();

if ($result->isSuccess()) {
    echo "Extracted to: " . $result->getData('destination');
}
```

## Password Protection

### password

```php
password(string $password): self
```

Sets a password for encrypted archives. Uses AES-256 encryption.

```php
// Create encrypted archive
$result = (new Zipper())
    ->add(['/path/to/sensitive.doc'])
    ->path('/path/to/output')
    ->password('secure_password_123')
    ->save('encrypted.zip');

// Extract encrypted archive
$result = (new Zipper())
    ->file('/path/to/encrypted.zip')
    ->path('/path/to/destination')
    ->password('secure_password_123')
    ->extract();
```

## Convenience Methods

### zipData

```php
zipData(array $paths, string $zipFilePath): bool
```

Convenience method to zip multiple directories and files recursively in one call.

```php
$success = (new Zipper())->zipData(
    [
        '/path/to/folder1',
        '/path/to/folder2',
        '/path/to/single_file.txt'
    ],
    '/path/to/output/complete_backup.zip'
);
```

## Result Object

| Method               | Type     | Description                                              |
| :------------------- | :------- | :------------------------------------------------------- |
| `isSuccess()`        | `bool`   | Whether the operation succeeded                          |
| `getMessage()`       | `string` | Status or error message                                  |
| `getData(?string k)` | `mixed`  | Returns operation data (e.g., `zip_path`, `destination`) |

## Examples

### Backup User Files

```php
function backupUserData(int $userId): string
{
    $userDir = storage_path("uploads/users/{$userId}");
    $backupDir = storage_path('backups');
    $filename = "user_{$userId}_" . date('Y-m-d') . '.zip';

    $result = (new Zipper())
        ->add(glob("{$userDir}/*"))
        ->path($backupDir)
        ->save($filename);

    if (!$result->isSuccess()) {
        throw new RuntimeException("Backup failed: {$result->getMessage()}");
    }

    return $result->getData('zip_path');
}
```

### Export Report Bundle

```php
function exportReportBundle(array $reportFiles): string
{
    $tempDir = storage_path('temp/exports');
    $filename = 'reports_' . date('Ymd_His') . '.zip';

    $result = (new Zipper())
        ->add($reportFiles)
        ->path($tempDir)
        ->password(config('exports.password'))
        ->save($filename);

    return $result->path;
}
```

### Restore from Backup

```php
function restoreFromBackup(string $backupPath, string $targetDir): bool
{
    // Clear target directory first
    FileSystem::delete($targetDir, true);
    FileSystem::mkdir($targetDir);

    $result = (new Zipper())
        ->file($backupPath)
        ->path($targetDir)
        ->extract();

    if (!$result->isSuccess()) {
        logger()->error("Restore failed", ['error' => $result->getMessage()]);
        return false;
    }

    return true;
}
```

### Recursive Directory Archive

```php
function archiveEntireDirectory(string $sourceDir, string $outputPath): bool
{
    return (new Zipper())->zipData([$sourceDir], $outputPath);
}

// Usage
archiveEntireDirectory(
    storage_path('logs'),
    storage_path('backups/logs_archive.zip')
);
```

## Related

- [FileSystem](filesystem-helper.md) - File operations
- [Paths](paths-helper.md) - Path helpers
- [Encryption](encryption.md) - File encryption
