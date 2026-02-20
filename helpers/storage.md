# Storage

The Storage component provides a powerful, consistent abstraction for file system operations across massive supported drivers. Whether you are working with a local disk, AWS S3, Azure Blob Storage, or Google Cloud Storage, the API remains exactly the same.

## Features

- **Unified API**: Write code once, deploy anywhere (Local, S3, Azure, GCS, FTP, SFTP).
- **Cloud Native**: First-class support for AWS S3, Cloudflare R2, MinIO, Azure Blob, and Google Cloud Storage.
- **Secure**: Features like Signed URLs (SAS/Presigned) for temporary access control.
- **Extensible**: Easily add custom drivers or extend existing ones.

## Configuration

The storage configuration is located at `App/Config/filesystems.php`.

### Default Disk

This option defines the default disk that will be used when you call storage static methods without specifying the disk name.

```php
'default' => env('FILESYSTEM_DISK', 'local'),
```

### Disks

You can configure as many disks as you like. Each disk represents a specific storage driver and location.

```php
'disks' => [
    'local' => [
        'driver' => 'local',
        'root' => storage_path('app'),
    ],

    's3' => [
        'driver' => 's3',
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION'),
        'bucket' => env('AWS_BUCKET'),
        'url' => env('AWS_URL'),
    ],

    'azure' => [
        'driver' => 'azure',
        'name' => env('AZURE_STORAGE_NAME'),
        'key' => env('AZURE_STORAGE_KEY'),
        'container' => env('AZURE_STORAGE_CONTAINER'),
    ],
    
    // ... other drivers (gcs, ftp, sftp, webdav, etc.)
],

/*
|--------------------------------------------------------------------------
| Temporary URL Configuration
|--------------------------------------------------------------------------
*/
'links' => [
    'threshold' => 1000000000, // Timestamps before this are treated as durations (seconds)
    'signed_route' => '/storage/signed/view', // Customizable route for local signed URLs
],
```

### S3 Compatible Services (R2, MinIO)

Since the `s3` driver is fully compliant with the AWS S3 API, you can easily use it with Cloudflare R2, MinIO, DigitalOcean Spaces, and others by configuring the `endpoint`.

**Cloudflare R2 Example:**

```php
'r2' => [
    'driver' => 's3',
    'key' => env('CLOUDFLARE_R2_ACCESS_KEY_ID'),
    'secret' => env('CLOUDFLARE_R2_SECRET_ACCESS_KEY'),
    'region' => 'auto',
    'bucket' => env('CLOUDFLARE_R2_BUCKET'),
    'endpoint' => 'https://<account_id>.r2.cloudflarestorage.com',
],
```

**MinIO Example:**

```php
'minio' => [
    'driver' => 's3',
    'key' => env('MINIO_ACCESS_KEY'),
    'secret' => env('MINIO_SECRET_KEY'),
    'region' => 'us-east-1',
    'bucket' => env('MINIO_BUCKET'),
    'endpoint' => 'http://minio:9000',
    'use_path_style_endpoint' => true,
],
```

## Basic Usage

The `Storage` manager allows you to interact with any of your configured disks.

```php
use Helpers\File\Storage\Storage;

// Use default disk
Storage::put('avatars/1.jpg', $content);

// Use specific disk
Storage::disk('s3')->put('avatars/1.jpg', $content);
```

### Retrieving Files

```php
$content = Storage::get('file.jpg');

if (Storage::exists('file.jpg')) {
    // ...
}
```

### Storing Files

```php
Storage::put('file.txt', 'Contents');

// With options (e.g. public visibility)
Storage::disk('s3')->put('file.txt', 'Contents', ['visibility' => 'public']);
```

### File URLs

```php
// Get public URL
$url = Storage::url('file.jpg');

// Get Temporary (Signed) URL
// Works seamlessly for Local (if configured), S3 (Presigned), and Azure (SAS)
$tempUrl = Storage::temporaryUrl('invoice.pdf', now()->addMinutes(5));
```

### File Manipulation

```php
Storage::copy('old/file.jpg', 'new/file.jpg');
Storage::move('from.jpg', 'to.jpg');
Storage::delete('file.jpg');
Storage::delete(['file1.jpg', 'file2.jpg']);
```

### Directories

```php
Storage::makeDirectory('photos');
Storage::deleteDirectory('photos');

// List files
$files = Storage::files('photos');
$allFiles = Storage::files('photos', recursive: true);
```

## Supported Drivers

The framework includes built-in support for the following drivers:

| Driver | Description | Configuration Keys |
| :--- | :--- | :--- |
| **local** | Local server filesystem. | `root` |
| **s3** | AWS S3, MinIO, Cloudflare R2, DigitalOcean Spaces. | `key`, `secret`, `region`, `bucket`, `endpoint` |
| **azure** | Microsoft Azure Blob Storage. | `name` (account), `key`, `container` |
| **gcs** | Google Cloud Storage (via S3 Interoperability). | `key`, `secret`, `bucket` |
| **ftp** | FTP Protocol. | `host`, `username`, `password`, `port`, `ssl` |
| **sftp** | SSH File Transfer Protocol. | `host`, `username`, `password`/`privateKey` |
| **webdav** | WebDAV Standard. | `baseUri`, `username`, `password` |
| **zip** | Zip Archive access. | `path` |
| **memory** | In-memory array (for testing). | None |
| **null** | Blackhole (discards writes). | None |

## Streaming Support

The storage system natively supports streaming for handling large files efficiently without loading the entire file into memory.

```php
// Write a large file from a stream
$stream = fopen('large-video.mp4', 'rb');
Storage::disk('s3')->writeStream('videos/1.mp4', $stream);

// Read a file as a stream (returns a resource)
$stream = Storage::disk('s3')->readStream('videos/1.mp4');
fpassthru($stream);

// Read a specific range (e.g. for resumable downloads or video seeking)
$stream = Storage::disk('s3')->readStream('videos/1.mp4', ['start' => 0, 'end' => 1048576]);
```

This ensures minimal memory usage even when processing gigabytes of data.

## Package Integration

The storage system is the backbone of the entire ecosystem. Here is how key packages utilize it:

### Media Package

The `Media` package uses the storage system to save uploaded files (images, documents).

- **Default Disk**: Controlled via `App/Config/media.php`.
- **Per-Upload**: You can specify a disk when uploading media.

```php
Media::upload($file, ['disk' => 's3']);
```

### Import Package

When uploading large CSV/Excel files for import, the `Import` package stores the temporary file on a storage disk before processing.

- **Cloud Support**: Configure `import.php` to use `s3` for uploading large files directly to cloud storage, which is ideal to avoid server timeouts.
- **R2 Support**: Simply use the `s3` driver with Cloudflare R2 credentials (defined in `filesystems.php`).

```php
Import::make(Importer::class)->disk('s3')->file($path)->execute();
```

### Export Package

Generated reports and exports are written to a storage disk.

- **Background Processing**: Queued exports write to the disk defined in `export.php` or `filesystems.php`.
- **Retention**: The cleanup command (`export:cleanup`) uses the storage driver to list and delete old files.

```php
Export::make(Exporter::class)->disk('azure')->csv()->queue()->execute();
```

## Security Best Practices

- **Never commit keys**: Always use `.env` for your storage credentials (`AWS_SECRET_ACCESS_KEY`, `AZURE_STORAGE_KEY`).
- **Use Private Buckets**: By default, keep your cloud buckets private and use `temporaryUrl()` to grant time-limited access to files.
- **Validate Inputs**: When using user input to construct file paths, ensure you validate and sanitize the path to prevent directory traversal.
