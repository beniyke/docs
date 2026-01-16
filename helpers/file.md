# File Helpers

The `Helpers\File` namespace provides a comprehensive suite of utilities for file system operations, caching, image manipulation, and archiving.

> Use the global helpers `filesystem()`, `cache()`, `mimes()`, and `image()` for quick access.

## Components

| Helper          | Description                                           | Documentation                      |
| :-------------- | :---------------------------------------------------- | :--------------------------------- |
| **Cache**       | File-based caching with TTL, tags, locking            | [cache](cache-helper.md)           |
| **FileSystem**  | Low-level file operations (read, write, copy, delete) | [filesystem](filesystem-helper.md) |
| **ImageHelper** | Image manipulation (resize, crop, effects)            | [image](image-helper.md)           |
| **Zipper**      | ZIP archive creation and extraction                   | [zipper](zipper-helper.md)         |
| **Paths**       | Application path helpers                              | [paths](paths-helper.md)           |
| **Mimes**       | MIME type detection                                   | [mimes](mimes-helper.md)           |
| **Lottery**     | Probabilistic execution and weighted selection        | [lottery](lottery-helper.md)       |
| **Benchmark**   | Execution time and memory profiling                   | [benchmark](benchmark-helper.md)   |

## Global Upload Helpers

These helpers provide convenient file upload validation and handling.

### validate_upload

```php
validate_upload(array $file, array $options): bool
```

Validates `$_FILES` data against type and size constraints.

| Option    | Description                              |
| :-------- | :--------------------------------------- |
| `type`    | Category: `image`, `document`, `archive` |
| `maxSize` | Maximum file size in bytes               |

```php
if (validate_upload($_FILES['avatar'], ['type' => 'image', 'maxSize' => 2097152])) {
    // File is valid
}
```

### upload_image

```php
upload_image(array $file, string $destination, int $maxSize = 5242880): string
```

Validates and moves an uploaded image file.

- **Allowed types**: jpg, jpeg, png, gif, webp, svg
- **Default max size**: 5MB
- **Returns**: Final file path

```php
$path = upload_image($_FILES['photo'], storage_path('uploads/photos'));
```

### upload_document

```php
upload_document(array $file, string $destination, int $maxSize = 10485760): string
```

Validates and moves an uploaded document file.

- **Allowed types**: pdf, doc, docx, xls, xlsx, ppt, pptx, txt, csv
- **Default max size**: 10MB
- **Returns**: Final file path

```php
$path = upload_document($_FILES['resume'], storage_path('uploads/documents'));
```

### upload_archive

```php
upload_archive(array $file, string $destination, int $maxSize = 52428800): string
```

Validates and moves an uploaded archive file.

- **Allowed types**: zip, rar, 7z, tar, gz
- **Default max size**: 50MB
- **Returns**: Final file path

```php
$path = upload_archive($_FILES['backup'], storage_path('uploads/archives'));
```

## Related

- [Encryption](encryption-helper.md) - File encryption
- [File Uploads](file-uploads.md) - Detailed upload handling guide
