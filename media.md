# Media

The Media package provides a versatile media library and file management system for the Anchor Framework. It supports image conversions, polymorphic attachments, and optimized storage handling.

## Features

- **Polymorphic Attachments**: Attach media to any model using the "HasMedia" trait.
- **Automated Conversions**: Automatically generate thumbnails and optimized versions of images.
- **Collections**: Organize media into logical groups (e.g., 'avatars', 'gallery').
- **Secure Uploads**: Built-in validation for file types, sizes, and MIME types.
- **Driver Support**: seamless integration with local storage or cloud disks.
- **Metadata Management**: Track dimensions, file size, and human-readable stats automatically.

### Vault Integration (Security)

For sensitive files that require encryption or strict quota management, `Media` integrates with the `Vault` package. Files uploaded via `Media` can be tracked in a user's `Vault` for compliance and billing.

```php
use Media\Media;
use Vault\Vault;

$media = Media::upload($file);

// Track the media item in the user's secure Vault
Vault::forAccount($user->id)
    ->trackUpload(Media::getPath($media), $media->size);
```

### Install the Package

```bash
php dock package:install Media --packages
```

This will automatically:

- Run the migration for Media tables.
- Register the `MediaServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/media.php`

```php
return [
    'disk' => 'local',
    'path' => 'media',
    'max_file_size' => 10485760, // 10MB

    'conversions' => [
        'thumbnail' => ['width' => 150, 'height' => 150],
        'medium' => ['width' => 600, 'height' => null],
    ],

    'auto_conversions' => true,
    'quality' => 85,
];
```

## Basic Usage

### Simple Upload

Use the `Media` facade for direct file handling:

```php
use Media\Media;

// From global upload array
$media = Media::upload($this->request->file('image'));

// From a local path or URL
$media = Media::uploadFromUrl('https://example.com/photo.jpg');

// Get URL
echo Media::url($media); // Original
echo Media::url($media, 'thumbnail'); // Conversion

// Accessing All Library Media
// Beyond polymorphic attachments, you can query media globally
$allMedia = Media::query()
    ->where('collection', 'identity_docs')
    ->latest()
    ->get();
```

## Use Cases

### Secure User Documents

Manage sensitive identity documents (Passports, ID Cards) that must be encrypted at rest and only accessible by authorized agents.

### Implementation

```php
use Media\Media;
use Permit\Permit;

public function uploadIdentityDoc()
{
    // 1. Upload to a secure collection
    $media = Media::upload($this->request->file('passport'), [
        'collection' => 'identity_docs',
        'metadata' => ['owner_id' => $user->id]
    ]);

    // 2. Integration: Track in Audit Trail
    Audit::make()
        ->event('media.id_upload')
        ->on($media)
        ->by($user)
        ->with('media_uuid', $media->uuid)
        ->log();

    return response()->json(['status' => 'uploaded']);
}
```

### Implementation (Action Pattern)

In Anchor, sensitive operations are encapsulated in Actions for reusability and clean controller logic.

```php
namespace App\Actions\Media;

use Media\Media;
use Permit\Permit;
use App\Models\User;
use Helpers\Http\Response;

class ViewSecureDocumentAction
{
    /**
     * View a secure document if the user has permission or is the owner.
     */
    public function execute(User $user, string $uuid): Response
    {
        $media = Media::findByUuid($uuid);
        
        if (!$media) {
            return response()->notFound();
        }

        
        if (!Permit::can($user, 'view-sensitive-media') && ($media->metadata['owner_id'] ?? null) !== $user->id) {
            return response()->forbidden();
        }

        return response()->file(Media::getPath($media));
    }
}
```

### Usage in Controller

```php
use Helpers\Http\Response;

public function show(string $uuid, ViewSecureDocumentAction $action): Response
{
    return $action->execute($this->auth->user(), $uuid);
}
```

### Sample Data (JSON)

State of a secure media record:

```json
{
  "uuid": "med_secure_abcdef",
  "filename": "passport_scan.pdf",
  "mime_type": "application/pdf",
  "collection": "identity_docs",
  "disk": "s3",
  "metadata": {
    "owner_id": 123,
    "encrypted": true
  }
}
```

## Package Integrations

### Vault Package (Quota & Storage)

Files uploaded via `Media` can be automatically monitored by `Vault` to enforce storage quotas.

```php
use Media\Media;
use Vault\Vault;

// Upload media and track its size in the user's secure vault
$media = Media::upload($file);

Vault::forAccount($user->id)
    ->trackUpload(Media::getPath($media), $media->size);

// Check if user has space before next upload
if (Vault::forAccount($user->id)->isFull()) {
    throw new QuotaExceededException("Storage full.");
}
```

### Audit Package (Traceability)

Every media upload, deletion, or restoration is logged by the `Audit` package, providing a clear chain of custody.

```php
use Audit\Audit;
use Media\Media;

// Manually logging a sensitive media access
public function accessSensitiveMedia($media)
{
    Audit::make()
        ->event('media.accessed')
        ->on($media)
        ->with('user_agent', request()->userAgent())
        ->log();

    return Media::url($media);
}
```

## Model Integration

Add the `HasMedia` trait to any model:

```php
use Media\Traits\HasMedia;

class Post extends BaseModel
{
    use HasMedia;
}
```

### Usage with Models

```php
$post = Post::find(1);

// Attach a file to a collection
$post->addMedia($uploadedFile, 'featured_image');

// Retrieve media
$image = $post->getFirstMedia('featured_image');
echo Media::url($image);

// Clear a collection
$post->clearMediaCollection('temp');
```

## Advanced Features

### Custom Conversions

While global conversions are set in config, you can define per-model conversions:

```php
public function registerMediaConversions(): void
{
    Media::addConversion('square_thumb')
        ->width(200)
        ->height(200);
}
```

### Global Analytics

Track disk usage and media volume:

```php
$analytics = Media::analytics();

// Disk usage by type (image, video, doc)
$usage = $analytics->getUsageByType();

// Volume trends
$trends = $analytics->getUploadTrends(30);
```

## Service API Reference

### Media (Facade)

| Method                    | Description                                       |
| :------------------------ | :------------------------------------------------ |
| `upload($file, $options)` | Handles a raw file upload.                        |
| `uploadFromUrl($url)`     | Fetches and stores a file from a remote URL.      |
| `url($media, $conv)`      | Returns the public URL for a specific conversion. |
| `getPath($media, $conv)`  | Returns the local file path.                      |
| `delete($media)`          | Removes the file and its database record.         |
| `analytics()`             | Returns the `MediaAnalytics` service.             |

### HasMedia (Trait)

| Method                         | Description                                     |
| :----------------------------- | :---------------------------------------------- |
| `addMedia($file, $collection)` | Attaches a file to the model.                   |
| `getMedia($collection)`        | Returns a collection of media models.           |
| `getFirstMedia($collection)`   | Returns the first media record in a collection. |
| `clearMediaCollection($name)`  | Deletes all media in a specific collection.     |

### Media (Model)

| Method           | Type      | Description                      |
| :--------------- | :-------- | :------------------------------- |
| `isImage()`      | `boolean` | True if the file is an image.    |
| `getHumanSize()` | `string`  | Formatted size (e.g., "1.2 MB"). |

## Troubleshooting

| Error/Log                     | Cause                                       | Solution                                   |
| :---------------------------- | :------------------------------------------ | :----------------------------------------- |
| `FileTypeNotAllowedException` | The file MIME type is not in the whitelist. | Update `allowed_types` in `media.php`.     |
| `FileSizeExceededException`   | The file size exceeds the maximum limit.    | Update `max_file_size` in `media.php`.     |
| "Unable to generate thumbnail" | GD or ImageMagick is not installed.         | Install the required PHP extension.        |
| "File not found"              | Record exists but file is missing on disk.  | Check the storage disk/path configuration. |

## Security Best Practices

- **MIME Verification**: Never trust the file extension; always verify the actual MIME type (handled automatically by the package).
- **Path Obfuscation**: Avoid using user-supplied names for physical file storage to prevent path traversal attacks.
- **Hotlinking Protection**: Configure your web server (Nginx/Apache) to prevent private media from being accessed without authentication.
