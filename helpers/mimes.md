# Mimes Helper

The `Helpers\File\Mimes` class identifies MIME types from file extensions and vice versa. It contains an extensive mapping of common file types.

> Use the `mimes()` global function for convenient access.

```php
// Get MIME type from extension
$type = Mimes::guessTypeFromExtension('pdf');
// application/pdf

// Get extension from MIME type
$ext = Mimes::guessExtensionFromType('image/jpeg');
// jpg
```

## Methods

### guessTypeFromExtension

```php
static guessTypeFromExtension(string $extension): ?string
```

Returns the primary MIME type for a given file extension.

```php
Mimes::guessTypeFromExtension('jpg');   // image/jpeg
Mimes::guessTypeFromExtension('pdf');   // application/pdf
Mimes::guessTypeFromExtension('mp4');   // video/mp4
Mimes::guessTypeFromExtension('docx');  // application/vnd.openxmlformats-officedocument.wordprocessingml.document
Mimes::guessTypeFromExtension('xyz');   // null (unknown)
```

### guessExtensionFromType

```php
static guessExtensionFromType(string $type, ?string $proposed_extension = null): ?string
```

Returns the most likely file extension for a MIME type.

| Parameter             | Description                                          |
| :-------------------- | :--------------------------------------------------- |
| `$type`               | The MIME type to look up                             |
| `$proposed_extension` | Optional hint to prefer if multiple extensions match |

```php
Mimes::guessExtensionFromType('image/jpeg');           // jpg
Mimes::guessExtensionFromType('image/jpeg', 'jpeg');   // jpeg (prefers hint)
Mimes::guessExtensionFromType('application/pdf');      // pdf
Mimes::guessExtensionFromType('text/plain');           // txt
```

### extractExtension

```php
static extractExtension(string $string): string
```

Extracts the file extension from a filename or path.

```php
Mimes::extractExtension('document.pdf');           // pdf
Mimes::extractExtension('/path/to/image.jpg');     // jpg
Mimes::extractExtension('archive.tar.gz');         // gz
```

## Supported Types

The class includes mappings for hundreds of file types across these categories:

### Images

| Extension     | MIME Type       |
| :------------ | :-------------- |
| `jpg`, `jpeg` | `image/jpeg`    |
| `png`         | `image/png`     |
| `gif`         | `image/gif`     |
| `webp`        | `image/webp`    |
| `svg`         | `image/svg+xml` |
| `ico`         | `image/x-icon`  |
| `bmp`         | `image/bmp`     |
| `tiff`        | `image/tiff`    |

### Documents

| Extension | MIME Type                                                                   |
| :-------- | :-------------------------------------------------------------------------- |
| `pdf`     | `application/pdf`                                                           |
| `doc`     | `application/msword`                                                        |
| `docx`    | `application/vnd.openxmlformats-officedocument.wordprocessingml.document`   |
| `xls`     | `application/vnd.ms-excel`                                                  |
| `xlsx`    | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`         |
| `ppt`     | `application/vnd.ms-powerpoint`                                             |
| `pptx`    | `application/vnd.openxmlformats-officedocument.presentationml.presentation` |
| `txt`     | `text/plain`                                                                |
| `csv`     | `text/csv`                                                                  |

### Archives

| Extension | MIME Type                      |
| :-------- | :----------------------------- |
| `zip`     | `application/zip`              |
| `rar`     | `application/x-rar-compressed` |
| `7z`      | `application/x-7z-compressed`  |
| `tar`     | `application/x-tar`            |
| `gz`      | `application/gzip`             |

### Audio/Video

| Extension | MIME Type         |
| :-------- | :---------------- |
| `mp3`     | `audio/mpeg`      |
| `wav`     | `audio/wav`       |
| `ogg`     | `audio/ogg`       |
| `mp4`     | `video/mp4`       |
| `webm`    | `video/webm`      |
| `avi`     | `video/x-msvideo` |
| `mov`     | `video/quicktime` |

### Web

| Extension | MIME Type                |
| :-------- | :----------------------- |
| `html`    | `text/html`              |
| `css`     | `text/css`               |
| `js`      | `application/javascript` |
| `json`    | `application/json`       |
| `xml`     | `application/xml`        |

## Examples

### Validate Upload MIME Type

```php
function validateImageUpload(array $file): bool
{
    $extension = Mimes::extractExtension($file['name']);
    $mimeType = Mimes::guessTypeFromExtension($extension);

    $allowedTypes = [
        'image/jpeg',
        'image/png',
        'image/gif',
        'image/webp'
    ];

    return in_array($mimeType, $allowedTypes);
}
```

### Set Content-Type Header

```php
function serveFile(string $path): void
{
    $extension = Mimes::extractExtension($path);
    $mimeType = Mimes::guessTypeFromExtension($extension) ?? 'application/octet-stream';

    header("Content-Type: {$mimeType}");
    header("Content-Length: " . filesize($path));

    readfile($path);
}
```

### Generate Safe Filename

```php
function generateSafeFilename(string $originalName, string $mimeType): string
{
    // Get correct extension for the actual MIME type
    $extension = Mimes::guessExtensionFromType($mimeType);

    // Generate unique filename with correct extension
    return uniqid() . '.' . $extension;
}
```

### Detect File Category

```php
function getFileCategory(string $filename): string
{
    $extension = Mimes::extractExtension($filename);
    $mimeType = Mimes::guessTypeFromExtension($extension);

    if (!$mimeType) {
        return 'unknown';
    }

    if (str_starts_with($mimeType, 'image/')) {
        return 'image';
    }

    if (str_starts_with($mimeType, 'video/')) {
        return 'video';
    }

    if (str_starts_with($mimeType, 'audio/')) {
        return 'audio';
    }

    if (in_array($mimeType, ['application/pdf', 'application/msword'])) {
        return 'document';
    }

    if (in_array($mimeType, ['application/zip', 'application/x-rar-compressed'])) {
        return 'archive';
    }

    return 'other';
}
```

## Related

- [FileSystem](filesystem-helper.md) - File operations
- [File Uploads](file-uploads.md) - Upload handling
- [Image](image-helper.md) - Image manipulation
