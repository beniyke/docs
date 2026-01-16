# Image Helper

The `Helpers\File\ImageHelper` class provides a fluent interface for image manipulation, powered by Intervention Image. It supports resizing, cropping, effects, filters, and watermarking.

> Use the `image()` global function for convenient access.

```php
// Resize and save an image
image('/path/to/photo.jpg')
    ->width(800)
    ->height(600)
    ->resize()
    ->save('/path/to/output', 'thumbnail.jpg', 85);
```

## Configuration

The image helper is configured in `App/Config/image.php`. You can set the driver (GD or Imagick), define resizing presets, and set default watermark settings.

```php
return [
    'driver' => 'gd', // 'gd' or 'imagick'
    'presets' => [
        'small' => ['width' => 200, 'height' => 200],
        'medium' => ['width' => 600, 'height' => 400],
    ],
    'watermark' => [
        'path' => 'img/logo.png', // Relative to public root
        'setting' => [
            'width' => 50,
            'height' => 50,
            'position' => 'bottom-right',
            'opacity' => 90,
            'padding' => ['x' => 10, 'y' => 10],
        ],
    ],
];
```

## Loading Images

### image

```php
image(string $path): self
```

Loads an image file for manipulation.

```php
$img = image('/path/to/source.jpg');
```

### imageWidth / imageHeight

```php
imageWidth(): int
imageHeight(): int
```

Returns the current dimensions of the loaded image.

```php
$img = image('/path/to/photo.jpg');
echo "Dimensions: {$img->imageWidth()}x{$img->imageHeight()}";
```

## Resizing

### width / height

```php
width(int $width): self

height(int $height): self
```

Sets the target dimensions for resize operations.

```php
image($path)
    ->width(1200)
    ->height(800)
    ->resize()
    ->save($dest, 'resized.jpg');
```

### resize

```php
resize(?string $scale = null): self
```

Resizes the image. If `$scale` is provided, it uses dimensions defined in `App/Config/image.php`. Otherwise, it uses manual dimensions set via `width()` and `height()`.

| Preset (Config) | Default Dimensions |
| :-------------- | :----------------- |
| `small`         | 200x200            |
| `medium`        | 600x400            |

```php
// Using preset
image($path)->resize('medium')->save($dest, 'medium.jpg');

// Using custom dimensions
image($path)
    ->width(500)
    ->height(300)
    ->resize()
    ->save($dest, 'custom.jpg');
```

## Cropping

### crop

```php
crop(string $crop = 'fit'): self
```

Crops the image using different strategies.

| Strategy  | Description                                  |
| :-------- | :------------------------------------------- |
| `fit`     | Fit within dimensions, maintain aspect ratio |
| `fill`    | Fill dimensions, crop overflow               |
| `stretch` | Stretch to exact dimensions (distorts)       |
| `pad`     | Pad to dimensions with background color      |

```php
image($path)
    ->width(400)
    ->height(400)
    ->crop('fill')
    ->save($dest, 'square.jpg');
```

### fit

```php
fit(): self
```

Alias for `crop('fit')`. Fits the image within the specified dimensions while maintaining aspect ratio.

```php
image($path)
    ->width(800)
    ->height(600)
    ->fit()
    ->save($dest, 'fitted.jpg');
```

## Orientation & Rotation

### orientation

```php
orientation(string $orientation = 'auto'): self
```

Adjusts the image orientation.

| Value  | Description                    |
| :----- | :----------------------------- |
| `auto` | Auto-orient based on EXIF data |
| `0`    | No rotation                    |
| `90`   | Rotate 90° clockwise           |
| `180`  | Rotate 180°                    |
| `270`  | Rotate 270° clockwise          |

```php
// Fix orientation from camera EXIF
image($path)->orientation('auto')->save($dest, 'fixed.jpg');
```

### rotate

```php
rotate(string $rotate): self
```

Rotates the image by the specified degrees. Accepts values from -270 to 270.

```php
image($path)->rotate('45')->save($dest, 'tilted.jpg');
```

### flip

```php
flip(string $flip): self
```

Flips the image.

| Value  | Description       |
| :----- | :---------------- |
| `h`    | Flip horizontally |
| `v`    | Flip vertically   |
| `both` | Flip both axes    |

```php
image($path)->flip('h')->save($dest, 'mirrored.jpg');
```

## Color Adjustments

### brightness

```php
brightness(string $brightness): self
```

Adjusts brightness. Values range from -100 to +100, where 0 is no change.

```php
image($path)->brightness('20')->save($dest, 'brighter.jpg');
```

### contrast

```php
contrast(string $contrast): self
```

Adjusts contrast. Values range from -100 to +100, where 0 is no change.

```php
image($path)->contrast('15')->save($dest, 'more_contrast.jpg');
```

### gamma

```php
gamma(float $gamma): self
```

Adjusts gamma. Values range from 0.1 to 9.99.

```php
image($path)->gamma(1.5)->save($dest, 'gamma_adjusted.jpg');
```

### opacity

```php
opacity(int $opacity): self
```

Sets the image opacity. Values range from 1 to 100.

```php
image($path)->opacity(50)->save($dest, 'semi_transparent.png');
```

### invert

```php
invert(): self
```

Inverts all colors in the image (negative effect).

```php
image($path)->invert()->save($dest, 'inverted.jpg');
```

## Effects

### sharpen

```php
sharpen(int $sharpen): self
```

Sharpens the image. Values range from 0 to 100.

```php
image($path)->sharpen(25)->save($dest, 'sharpened.jpg');
```

### blur

```php
blur(int $blur = 0): self
```

Applies a blur effect. Values range from 0 to 100.

```php
image($path)->blur(10)->save($dest, 'blurred.jpg');
```

### pixelate

```php
pixelate(string $pixel): self
```

Applies a pixelation effect. Values range from 0 to 1000.

```php
image($path)->pixelate('20')->save($dest, 'pixelated.jpg');
```

### filter

```php
filter(string $filter): self
```

Applies a color filter.

| Filter      | Description          |
| :---------- | :------------------- |
| `greyscale` | Convert to grayscale |
| `sepia`     | Apply sepia tone     |

```php
image($path)->filter('greyscale')->save($dest, 'bw.jpg');
image($path)->filter('sepia')->save($dest, 'vintage.jpg');
```

## Watermarking

### watermark

```php
watermark(array $config = []): self
```

Inserts a watermark image.

| Config Key | Description               | Default |
| :--------- | :------------------------ | :------ |
| `opacity`  | Watermark opacity (0-100) | `50`    |

```php
image($path)
    ->watermark([
        'watermark' => [
            'path' => '/path/to/logo.png',
            'setting' => [
                'width' => 50,
                'height' => 50,
                'position' => 'bottom-right',
                'opacity' => 90,
                'padding' => ['x' => 10, 'y' => 10],
            ],
        ],
    ])
    ->save($dest, 'watermarked.jpg');
```

> [!NOTE]
> If no configuration is provided, it uses the defaults from `App/Config/image.php`.

## Output

### encode

```php
encode(string $format, int $quality = 90): self
```

Encodes the image to a specific format.

| Format | Description |
| :----- | :---------- |
| `jpg`  | JPEG format |
| `png`  | PNG format  |
| `gif`  | GIF format  |
| `webp` | WebP format |

```php
$encoded = image($path)->encode('webp', 80);
```

### save

```php
save(string $save_to, string $save_as, int $quality = 90): bool
```

Saves the manipulated image to disk.

| Parameter  | Description           |
| :--------- | :-------------------- |
| `$save_to` | Destination directory |
| `$save_as` | Output filename       |
| `$quality` | JPEG quality (1-100)  |

```php
image($path)
    ->width(800)
    ->resize()
    ->save('/path/to/output', 'resized.jpg', 85);
```

## Examples

### Thumbnail Generation

```php
function generateThumbnail(string $source, string $destDir): string
{
    $filename = 'thumb_' . basename($source);

    image($source)
        ->width(150)
        ->height(150)
        ->crop('fill')
        ->save($destDir, $filename, 80);

    return $destDir . '/' . $filename;
}
```

### Profile Picture Processing

```php
function processProfilePicture(string $uploadedPath, int $userId): array
{
    $destDir = storage_path("uploads/avatars/{$userId}");

    // Generate multiple sizes
    $sizes = [
        'large' => [400, 400],
        'medium' => [200, 200],
        'small' => [50, 50],
    ];

    $paths = [];

    foreach ($sizes as $name => [$width, $height]) {
        image($uploadedPath)
            ->width($width)
            ->height($height)
            ->crop('fill')
            ->save($destDir, "{$name}.jpg", 85);

        $paths[$name] = "{$destDir}/{$name}.jpg";
    }

    return $paths;
}
```

### Image Optimization

```php
function optimizeImage(string $path): void
{
    image($path)
        ->orientation('auto')  // Fix EXIF rotation
        ->width(2048)          // Max width
        ->resize()
        ->sharpen(10)          // Slight sharpening
        ->save(dirname($path), basename($path), 85);
}
```

## Related

- [FileSystem](filesystem-helper.md) - File operations
- [Mimes](mimes-helper.md) - MIME type detection
- [File Uploads](file-uploads.md) - Upload handling
