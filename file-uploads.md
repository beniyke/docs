# File Uploads

Comprehensive guide to handling file uploads in the Anchor framework using the `FileHandler` class and `Request` methods.

## FileHandler Class

The `Helpers\Http\FileHandler` class provides robust file upload handling with validation, security checks, and file manipulation.

### Basic Usage

```php
use Helpers\Http\FileHandler;

// Get uploaded file from request
$file = new FileHandler($_FILES['avatar']);

// Or from Request class
$fileData = request()->file('avatar');
$file = new FileHandler($fileData);
```

## File Information Methods

### Get File Details

```php
use Helpers\Http\FileHandler;

$file = new FileHandler($_FILES['document']);

// Get original filename
$originalName = $file->getClientOriginalName();
// Returns: "report.pdf"

// Get MIME type
$mimeType = $file->getClientMimeType();
// Returns: "application/pdf"

// Get file extension (from MIME type)
$extension = $file->getExtension();
// Returns: "pdf"

// Get original extension (from filename)
$originalExt = $file->getClientOriginalExtension();
// Returns: "pdf"

// Get file size in bytes
$size = $file->getClientSize();
// Returns: 1024000

// Get temporary path
$tempPath = $file->getpathName();
// Returns: "/tmp/phpXXXXXX"
```

## File Validation

### Check Upload Status

```php
// Check if upload was successful
if ($file->isValid()) {
    // File uploaded successfully
}

// Check if file has errors
if ($file->hasError()) {
    $errorMessage = $file->getErrorMessage();
    // Returns human-readable error like "File exceeds maximum size"
}

// Check if file is empty
if ($file->isEmpty()) {
    // No file was uploaded
}

// Get error code
$errorCode = $file->getError();
// Returns: UPLOAD_ERR_OK (0) if successful
```

### Validation with Options

```php
// Validate file with specific criteria
$isValid = $file->validate([
    'maxSize' => '5mb',
    'extensions' => ['jpg', 'png', 'pdf'],
    'mimeTypes' => ['image/jpeg', 'image/png', 'application/pdf'],
]);

if (!$isValid) {
    $error = $file->getValidationError();
    echo "Validation failed: {$error}";
}
```

### Custom Validator

```php
use Helpers\File\FileUploadValidator;

// Create custom validator
$validator = $file->createValidator([
    'maxSize' => '10mb',
    'extensions' => ['docx', 'xlsx', 'pptx']
]);

$file->validateWith($validator);
```

## Moving Files

### Basic Move

```php
// Move to destination with optional custom name
$success = $file->move('storage/uploads', 'user_123_avatar.jpg');

if ($success) {
    echo "File uploaded successfully";
}

// Move with automatic name sanitization
$file->move('storage/documents');
// Uses original filename, sanitized
```

### Secure Move

Validates and moves file securely:

```php
// Move with validation
$path = $file->moveSecurely('storage/uploads', [
    'maxSize' => '2mb',
    'extensions' => ['jpg', 'png'],
    'mimeTypes' => ['image/jpeg', 'image/png']
]);

if ($path !== '') {
    echo "File validated and moved successfully to: {$path}";
} else {
    echo "Validation failed: " . $file->getValidationError();
}

// Throw exception on failure
try {
    $file->moveSecurely('storage/uploads', $options, true);
} catch (Exception $e) {
    echo "Upload failed: " . $e->getMessage();
}
```

## Using with Request Class

### Get Uploaded Files

```php
use Helpers\Http\Request;

class UploadController
{
    public function __construct(
        private Request $request
    ) {}

    public function upload()
    {
        // Check if request has files
        if (!$this->request->hasFile()) {
            return response()->json(['error' => 'No file uploaded'], 400);
        }

        // Get specific file
        $avatarData = $this->request->file('avatar');
        $avatar = new FileHandler($avatarData);

        // Process upload
        if ($avatar->isValid()) {
            $avatar->moveSecurely('storage/avatars', [
                'maxSize' => '1mb',
                'extensions' => ['jpg', 'png']
            ]);
        }
    }
}
```

### Multiple Files

```php
// Get all uploaded files
$files = $this->request->file();

foreach ($files as $key => $fileData) {
    $file = new FileHandler($fileData);

    if ($file->isValid()) {
        $file->move('storage/uploads');
    }
}

// Handle multiple files with same name (e.g., documents[])
$documents = $this->request->file('documents');

if (is_array($documents)) {
    foreach ($documents as $docData) {
        $doc = new FileHandler($docData);
        $doc->moveSecurely('storage/documents', $options);
    }
}
```

## Complete Upload Example

```php
use Helpers\Http\FileHandler;
use Helpers\Http\Request;

class ProfileController
{
    public function __construct(
        private Request $request
    ) {}

    public function uploadAvatar()
    {
        // Validate request has file
        if (!$this->request->hasFile()) {
            return response()->json([
                'error' => 'No file provided'
            ], 400);
        }

        // Get file
        $fileData = $this->request->file('avatar');
        $file = new FileHandler($fileData);

        // Check basic validity
        if (!$file->isValid()) {
            return response()->json([
                'error' => $file->getErrorMessage()
            ], 400);
        }

        // Generate unique filename
        $userId = $this->auth->user()->id;
        $extension = $file->getExtension();
        $filename = "avatar_{$userId}." . $extension;

        // Validate and move securely
        try {
            $filename = $file->moveSecurely('storage/avatars', [
                'type' => 'image',
                'maxSize' => '2mb'
            ], true);  // Throw on error

            // Update user avatar in database
            $user = $this->auth->user();
            $user->avatar = basename($filename);
            $user->save();

            return response()->json([
                'success' => true,
                'filename' => $filename,
                'url' => url('storage/avatars/' . $filename)
            ]);

        } catch (Exception $e) {
            return response()->json([
                'error' => 'Upload failed: ' . $e->getMessage()
            ], 400);
        }
    }
}
```

## File Upload Errors

Common upload errors and their handling:

```php
switch ($file->getError()) {
    case UPLOAD_ERR_OK:
        // Success
        break;

    case UPLOAD_ERR_INI_SIZE:
    case UPLOAD_ERR_FORM_SIZE:
        $error = "File exceeds maximum size";
        break;

    case UPLOAD_ERR_PARTIAL:
        $error = "File was only partially uploaded";
        break;

    case UPLOAD_ERR_NO_FILE:
        $error = "No file was uploaded";
        break;

    case UPLOAD_ERR_NO_TMP_DIR:
        $error = "Missing temporary folder";
        break;

    case UPLOAD_ERR_CANT_WRITE:
        $error = "Failed to write file to disk";
        break;

    case UPLOAD_ERR_EXTENSION:
        $error = "Upload stopped by PHP extension";
        break;
}

// Or use built-in error messages
$error = $file->getErrorMessage();
```

## Maximum File Size

```php
// Get PHP's max upload size
$maxSize = FileHandler::getMaxFilesize();
// Returns size in bytes based on php.ini settings

// Display to users
$maxSizeMB = round($maxSize / 1024 / 1024, 2);
echo "Maximum upload size: {$maxSizeMB}MB";
```

## Security Best Practices

### Validate File Types

```php
// ✅ Good - Whitelist allowed types
$file->validate([
    'extensions' => ['jpg', 'png', 'pdf'],
    'mimeTypes' => ['image/jpeg', 'image/png', 'application/pdf']
]);

// ❌ Bad - Accepting all file types
$file->move('storage/uploads');  // No validation
```

### Sanitize Filenames

```php
// FileHandler automatically sanitizes filenames
// Removes: ../, ./, null bytes, special characters

// Original: "../../../etc/passwd"
// Sanitized: "etc-passwd"

// Manual sanitization if needed
$safeName = $file->_sanitize($unsafeName);
```

### Store Outside Web Root

```php
// ✅ Good - Store outside public directory
$file->move('/var/app/storage/uploads');

// ❌ Bad - Storing in public directory
$file->move('public/uploads');  // Directly accessible
```

### Validate File Size

Prevent resource exhaustion:

```php
$file->validate([
    'maxSize' => '5mb'
]);
```

## Complete Method Reference

| Method                                                    | Description                             |
| --------------------------------------------------------- | --------------------------------------- |
| `__construct(array $file)`                                | Initialize with file data from $\_FILES |
| `getClientOriginalName()`                                 | Get original filename                   |
| `getClientMimeType()`                                     | Get MIME type                           |
| `getClientOriginalExtension()`                            | Get file extension from filename        |
| `getExtension()`                                          | Get extension from MIME type            |
| `getClientSize()`                                         | Get file size in bytes                  |
| `getpathName()`                                           | Get temporary file path                 |
| `getError()`                                              | Get upload error code                   |
| `isValid()`                                               | Check if upload succeeded               |
| `hasError()`                                              | Check if upload has errors              |
| `isEmpty()`                                               | Check if no file uploaded               |
| `getErrorMessage()`                                       | Get human-readable error                |
| `move(string $path, ?string $name)`                       | Move file to destination                |
| `validate(array $options)`                                | Validate with criteria                  |
| `validateWith(FileUploadValidator $validator)`            | Validate with custom validator          |
| `moveSecurely(string $dest, array $options, bool $throw)` | Validate and move (returns path)        |
| `getValidationError()`                                    | Get validation error message            |
| `createValidator(array $options)`                         | Create FileUploadValidator instance     |
| `getMaxFilesize()` static                                 | Get PHP max upload size                 |

## Related Documentation

- [Request](request-helper.md) - Request->file() methods
- [Validation](validation-helper.md) - File upload validation rules
- [Security](security.md) - File upload security
