# FileSystem Helper

The `Helpers\File\FileSystem` class provides static methods for low-level file system operations including reading, writing, copying, moving, and deleting files and directories.

> Use the `filesystem()` global function for convenient access.

```php
// Check if file exists
if (FileSystem::exists('/path/to/file.txt')) {
    $content = FileSystem::get('/path/to/file.txt');
}

// Write content to file
FileSystem::put('/path/to/output.txt', $content);
```

## Existence & Type Checks

### exists

```php
static exists(string $filename): bool
```

Returns `true` if the file or directory exists.

```php
if (FileSystem::exists($configPath)) {
    $config = require $configPath;
}
```

### isFile

```php
static isFile(string $path): bool
```

Returns `true` if the path points to a file (not a directory).

### isDir

```php
static isDir(string $path): bool
```

Returns `true` if the path points to a directory.

### isReadable

```php
static isReadable(string $path): bool
```

Returns `true` if the path is readable by the current process.

```php
if (FileSystem::isReadable($logFile)) {
    $logs = FileSystem::get($logFile);
}
```

### isWithin

```php
static isWithin(string $path, string $basePath): bool
```

Returns `true` if the path is physically located within the specified base directory.

```php
if (FileSystem::isWithin($userPath, storage_path('uploads'))) {
    // Safe to process
}
```

## Reading Files

### get

```php
static get(string $path, bool $lock = false): string
```

Reads and returns the contents of a file.

| Parameter | Description                                    |
| :-------- | :--------------------------------------------- |
| `$path`   | Absolute path to the file                      |
| `$lock`   | If `true`, acquire a shared lock while reading |

```php
$content = FileSystem::get('/path/to/file.txt');

// With locking for concurrent access
$content = FileSystem::get('/path/to/shared.txt', true);
```

### read

```php
static read(string $path): string
```

Alias for `get()`. Reads file contents without locking.

## Writing Files

### put

```php
static put(string $path, string $content, bool $lock = false): bool
```

Writes content to a file. Creates the file if it doesn't exist.

| Parameter  | Description                                        |
| :--------- | :------------------------------------------------- |
| `$path`    | Absolute path to the file                          |
| `$content` | Content to write                                   |
| `$lock`    | If `true`, acquire an exclusive lock while writing |

```php
FileSystem::put('/path/to/output.txt', 'Hello World');

// With exclusive lock
FileSystem::put('/path/to/shared.txt', $data, true);
```

### write

```php
static write(string $path, string $content): bool
```

Alias for `put()`. Writes content without locking.

### replace

```php
static replace(string $path, string $content): void
```

Atomically replaces a file's contents. Uses a temporary file and rename to prevent race conditions and ensure the file is never in a partially-written state.

```php
// Safe for concurrent access
FileSystem::replace('/path/to/config.json', json_encode($config));
```

### append

```php
static append(string $path, string $data): bool
```

Appends content to the end of a file.

```php
FileSystem::append('/path/to/log.txt', "[" . date('Y-m-d H:i:s') . "] Event occurred\n");
```

### prepend

```php
static prepend(string $path, string $data): bool
```

Prepends content to the beginning of a file.

```php
FileSystem::prepend('/path/to/changelog.txt', "## v2.0.0\n- New feature\n\n");
```

## File Information

### size

```php
static size(string $path): int
```

Returns the file size in bytes.

```php
$bytes = FileSystem::size('/path/to/file.zip');
$megabytes = $bytes / 1024 / 1024;
```

### lastModified

```php
static lastModified(string $path): int
```

Returns the Unix timestamp of when the file was last modified.

```php
$timestamp = FileSystem::lastModified('/path/to/file.txt');
$date = date('Y-m-d H:i:s', $timestamp);
```

### hash

```php
static hash(string $path): string
```

Returns the MD5 hash of a file's contents.

```php
$hash = FileSystem::hash('/path/to/file.zip');

// Verify file integrity
if ($hash === $expectedHash) {
    echo "File is valid";
}
```

### extension

```php
static extension(string $path): string
```

Extracts and returns the file extension.

```php
$ext = FileSystem::extension('/path/to/image.jpg'); // 'jpg'
```

### permissions

```php
static permissions(string $path): string
```

Returns the 4-digit octal permission string.

```php
$perms = FileSystem::permissions('/path/to/script.sh'); // '0755'
```

## File Operations

### delete

```php
static delete(string $path, bool $preserve = false): bool
```

Deletes a file or recursively deletes a directory.

| Parameter   | Description                                              |
| :---------- | :------------------------------------------------------- |
| `$path`     | File or directory path                                   |
| `$preserve` | If `true`, delete contents but keep the directory itself |

```php
// Delete a file
FileSystem::delete('/path/to/file.txt');

// Delete a directory and all contents
FileSystem::delete('/path/to/temp_dir');

// Empty a directory but keep it
FileSystem::delete('/path/to/cache', true);
```

### move

```php
static move(string $path, string $target): bool
```

Moves a file or directory to a new location.

```php
FileSystem::move('/path/to/old.txt', '/path/to/new.txt');
FileSystem::move('/path/to/old_dir', '/path/to/new_dir');
```

### copy

```php
static copy(string $source, string $destination, ?int $flag = null): bool
```

Copies a file or recursively copies a directory.

```php
// Copy a file
FileSystem::copy('/path/to/source.txt', '/path/to/dest.txt');

// Copy a directory
FileSystem::copy('/path/to/source_dir', '/path/to/dest_dir');
```

### chmod

```php
static chmod(string $path, int $permission): bool
```

Changes file or directory permissions.

```php
FileSystem::chmod('/path/to/script.sh', 0755);
FileSystem::chmod('/path/to/config.php', 0644);
```

## Directory Operations

### mkdir

```php
static mkdir(string $path, int $permission = 0755, bool $recursive = true): bool
```

Creates a directory.

| Parameter     | Description                                         |
| :------------ | :-------------------------------------------------- |
| `$path`       | Directory path to create                            |
| `$permission` | Unix permission (default: 0755)                     |
| `$recursive`  | Create parent directories if needed (default: true) |

```php
FileSystem::mkdir('/path/to/new/nested/directory');

// Non-recursive (parent must exist)
FileSystem::mkdir('/path/to/single', 0755, false);
```

### contents

```php
static contents(string $path, ?int $flag = null): ?RecursiveIteratorIterator
```

Returns a recursive iterator for traversing directory contents.

```php
$iterator = FileSystem::contents('/path/to/directory');

foreach ($iterator as $file) {
    if ($file->isFile()) {
        echo $file->getPathname() . "\n";
    }
}
```

## Examples

### Safe Configuration Update

```php
function updateConfig(string $key, mixed $value): void
{
    $configPath = Paths::configPath('app.php');
    $config = require $configPath;

    $config[$key] = $value;

    $content = "<?php\n\nreturn " . var_export($config, true) . ";\n";

    // Atomic write prevents corruption
    FileSystem::replace($configPath, $content);
}
```

### Log Rotation

```php
function rotateLog(string $logFile, int $maxSize = 5242880): void
{
    if (!FileSystem::exists($logFile)) {
        return;
    }

    if (FileSystem::size($logFile) > $maxSize) {
        $timestamp = date('Y-m-d_His');
        $archivePath = str_replace('.log', "_{$timestamp}.log", $logFile);

        FileSystem::move($logFile, $archivePath);
    }
}
```

### Directory Cleanup

```php
function cleanupTempFiles(string $directory, int $maxAgeSeconds = 86400): int
{
    $deleted = 0;
    $now = time();

    $iterator = FileSystem::contents($directory);

    foreach ($iterator as $file) {
        if ($file->isFile()) {
            $age = $now - FileSystem::lastModified($file->getPathname());

            if ($age > $maxAgeSeconds) {
                FileSystem::delete($file->getPathname());
                $deleted++;
            }
        }
    }

    return $deleted;
}
```

## Related

- [Paths](paths-helper.md) - Application path helpers
- [Cache](cache-helper.md) - File-based caching
- [Mimes](mimes-helper.md) - MIME type detection
