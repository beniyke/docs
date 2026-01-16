# Request Validation

Anchor provides a robust, class-based approach to request validation for both web forms and API requests. By creating dedicated validation classes that extend `App\Core\BaseRequestValidation`, you can encapsulate validation logic, keep controllers clean, and easily reuse validation rules across different request types.

## Creating a Validation Class

To create a validation class, extend the `App\Core\BaseRequestValidation` abstract class. You are required to implement the `expected()` method, and optionally override `rules()` and `parameters()` methods.

```php
namespace App\Auth\Validations\Form;

use App\Core\BaseRequestValidation;

class LoginFormRequestValidation extends BaseRequestValidation
{
    /**
     * Define the expected form fields.
     * These fields MUST be present in the input data, or validation will fail immediately.
     */
    public function expected(): array
    {
        return ['email', 'password'];
    }

    /**
     * Define the validation rules for each field.
     */
    public function rules(): array
    {
        return [
            'email' => [
                'type' => 'email',
                'strict' => 'disposable,role', // NEW: Block disposable and role emails
            ],
            'password' => [
                'required' => true,
                'minlength' => 8,
            ],
        ];
    }

    /**
     * Define human-readable labels for REQUIRED fields.
     * Any field listed here is automatically treated as REQUIRED.
     */
    public function parameters(): array
    {
        return [
            'email' => 'Email Address',
            'password' => 'Password',
        ];
    }
}
```

## Required vs Optional Fields

By default, **all fields defined in `parameters()` are REQUIRED**. If a field is missing or empty, validation will fail automatically.

- **Explicitly Required**: You can add `'required' => true` for clarity, but it is not strictly necessary.
- **Optional Fields**: To make a field optional, you **MUST** explicitly set `'required' => false`.

```php
public function rules(): array
{
    return [
        // Required by default (implicit)
        'username' => ['type' => 'string'],

        // Explicitly Required (same behavior)
        'email' => ['required' => true, 'type' => 'email'],

        // Optional: Validation skipped if empty/null
        'description' => ['required' => false],
    ];
}
```

## Nested Field Validation

Anchor's validation system fully supports **dot notation**, allowing you to validate deep data structures (like JSON payloads) as easily as flat forms.

```php
public function expected(): array
{
    // Nested keys MUST exist in the input
    return [
        'user.email',
        'user.profile.age'
    ];
}

public function rules(): array
{
    return [
        'user.email' => [
            'type' => 'email',
            'required' => true
        ],
        'user.profile.age' => [
            'type' => 'numeric',
            'limit' => '18|100'
        ]
    ];
}
```

When validation fails, error keys will also use dot notation (e.g., `$validator->errors()['user.email']`), making it easy to map errors back to your UI or API response.

## Wildcard Array Validation

For lists of items, you can use the `*` wildcard to validate each item in an array.

```php
public function expected(): array
{
    return ['items'];
}

public function rules(): array
{
    return [
        'items.*.name' => ['required' => true, 'type' => 'string'],
        'items.*.quantity' => ['required' => true, 'type' => 'numeric', 'minlength' => 1]
    ];
}

public function parameters(): array
{
    return [
        'items.*.name' => 'Item Name',
        'items.*.quantity' => 'Quantity'
    ];
}
```

Anchor will automatically expand these rules for every item in the `items` array. If validation fails for the second item, the error key will be `items.1.name` and the label will automatically be formatted as "Item Name #2".

### Wildcard File Validation

You can also use wildcards to validate arrays of uploaded files.

```php
public function file(): array
{
    // Define the parameter name for the file array
    // 'file' method is preferred for file inputs to ensure proper merging
    return [
        'avatars.*' => 'User Avatar'
    ];
}

public function rules(): array
{
    return [
        'avatars.*' => [
            'required' => true,
            // Pass an array of extensions for multi-file rules
            'allowed_file_type' => ['png', 'jpg'],
            'allowed_file_size' => '2mb'
        ]
    ];
}
```

## Controller Integration

Validation classes are automatically resolved by the IoC container and injected into your controller methods.

```php
namespace App\Auth\Controllers;

use App\Auth\Validations\Form\LoginFormRequestValidation;
use App\Core\BaseController;
use Helpers\Http\Response;

class LoginController extends BaseController
{
    public function attempt(LoginFormRequestValidation $validator): Response
    {
        // 1. Run validation against POST data
        $validator->validate($this->request->post());

        // 2. Check for errors
        if ($validator->has_error()) {
            // Flash errors to session and redirect back
            $this->flash->error($validator->errors());
            return $this->response->redirect($this->request->fullRoute());
        }

        // 3. Retrieve validated data (returns a Data helper object)
        $data = $validator->validated();

        // ... proceed with login logic
    }
}
```

## Email Validation

Anchor provides comprehensive, production-ready email validation with **zero external dependencies**. Protect your application from disposable emails, role-based accounts, and enforce domain policies.

### Available Email Validation Rules

| Rule            | Description                                             | Example                                           |
| :-------------- | :------------------------------------------------------ | :------------------------------------------------ |
| `strict`        | Block disposable, role-based emails, verify MX, or SMTP | `'strict' => 'disposable,role,mx,smtp'`           |
| `allow_domains` | Whitelist specific domains (supports wildcards)         | `'allow_domains' => 'company.com,*.company.com'`  |
| `block_domains` | Blacklist specific domains (supports wildcards)         | `'block_domains' => 'competitor.com,*.spam.com'`  |
| `email_pattern` | Custom regex pattern matching                           | `'email_pattern' => '^[a-z0-9.]+@company\\.com$'` |

### Strict Validation

Block disposable emails, role-based accounts, verify DNS MX records, or perform SMTP mailbox verification:

```php
public function rules(): array
{
    return [
        'email' => [
            'type' => 'email',
            'strict' => 'disposable,smtp', // Block disposable and verify mailbox via SMTP
        ],
    ];
}
```

> SMTP verification (`smtp`) is thorough but can be slow (10s+) and may be blocked by some mail servers. Use it judiciously, preferably in background jobs or for high-value registrations.

### Domain Whitelisting

Only allow emails from specific domains:

```php
public function rules(): array
{
    return [
        'email' => [
            'type' => 'email',
            'allow_domains' => 'company.com,*.company.com',
        ],
    ];
}
```

**Wildcard patterns supported:**

- `company.com` - Exact match only
- `*.company.com` - Any subdomain (sales.company.com, hr.company.com)
- `*mail.com` - Any domain ending with "mail.com"

### Domain Blacklisting

Block specific domains:

```php
public function rules(): array
{
    return [
        'email' => [
            'type' => 'email',
            'block_domains' => 'competitor.com,spam.com,*.blocked.com',
        ],
    ];
}
```

### Pattern Matching

Enforce custom email formats with regex:

```php
public function rules(): array
{
    return [
        'email' => [
            'type' => 'email',
            'email_pattern' => '^[a-z0-9.]+@company\\.com$', // Only lowercase, numbers, dots
        ],
    ];
}
```

### Combined Email Validation

Use multiple rules together for maximum security:

```php
public function rules(): array
{
    return [
        'email' => [
            'type' => 'email',
            'strict' => 'disposable,role',
            'allow_domains' => '*.company.com,partner.com',
            'block_domains' => 'test.company.com',
        ],
    ];
}
```

**Validation order:**

1. Email format validation (`type => 'email'`)
2. Strict checks (disposable, role, mx)
3. Domain whitelist (`allow_domains`)
4. Domain blacklist (`block_domains`)
5. Pattern matching (`email_pattern`)

### Custom Error Messages

Provide user-friendly error messages:

```php
public function messages(): array
{
    return [
        'email' => [
            'strict' => 'Please use your company or personal email address',
            'allow_domains' => 'Only company email addresses are accepted',
            'block_domains' => 'This email domain is not allowed',
            'email_pattern' => 'Email must be in the format: name@company.com',
        ],
    ];
}
```

### Real-World Examples

#### Corporate Registration (Employees Only)

```php
class EmployeeRegistrationValidation extends BaseRequestValidation
{
    public function rules(): array
    {
        return [
            'email' => [
                'type' => 'email',
                'strict' => 'disposable,role',
                'allow_domains' => '*.company.com',
            ],
        ];
    }

    public function messages(): array
    {
        return [
            'email' => [
                'strict' => 'Please use your company email address',
                'allow_domains' => 'Only @company.com email addresses are allowed',
            ],
        ];
    }
}
```

#### Public Registration (Block Spam)

```php
class UserRegistrationValidation extends BaseRequestValidation
{
    public function rules(): array
    {
        return [
            'email' => [
                'type' => 'email',
                'strict' => 'disposable', // Block temporary emails
            ],
        ];
    }

    public function messages(): array
    {
        return [
            'email' => [
                'strict' => 'Temporary email addresses are not allowed. Please use a permanent email.',
            ],
        ];
    }
}
```

#### Contact Form (Block Competitors)

```php
class ContactFormValidation extends BaseRequestValidation
{
    public function rules(): array
    {
        return [
            'email' => [
                'type' => 'email',
                'strict' => 'disposable,role',
                'block_domains' => 'competitor.com,rival.com',
            ],
        ];
    }
}
```

### Configuration

Customize email validation behavior in `Config/email_validation.php`:

```php
return [
    // Add custom disposable domains
    'disposable_domains' => [
        'custom' => [
            'my-custom-disposable.com',
        ],
    ],

    // Add custom role prefixes
    'role_accounts' => [
        'custom' => [
            'customrole',
        ],
    ],

    // Global whitelist (always allowed)
    'global_whitelist' => [
        'trusted-partner.com',
    ],

    // Global blacklist (always blocked)
    'global_blacklist' => [
        'known-spam-domain.com',
    ],

    // DNS validation settings
    'dns_validation' => [
        'enabled' => true,
        'timeout' => 5, // seconds
        'graceful_fallback' => true, // Allow if DNS check fails
    ],
];
```

### Edge Cases Handled

- **Whitespace** - Automatically trimmed
- **Case sensitivity** - Domain matching is case-insensitive
- **Invalid regex** - Graceful error handling
- **DNS timeouts** - Configurable timeout with fallback
- **Unicode emails** - Properly handled
- **Special characters** - Supports `+` tags and dots in local part

### Performance Considerations

- **Disposable list**: 600+ domains, loaded on-demand
- **DNS validation**: Optional, 5-second timeout, use sparingly
- **Pattern matching**: Optimized regex with ReDoS protection
- **Caching**: Consider caching validation results for high-traffic applications

---

## Class Reference: BaseRequestValidation

The `BaseRequestValidation` class provides numerous methods you can override to customize behavior for both web form and API request validation.

### Core Configuration (Required/Abstract)

#### expected

```php
expected(): array
```

Lists the field keys that **must** be present in the input data. If any are missing, validation fails immediately.

- **Use Case**: Shielding your logic from missing POST keys that would cause PHP warnings or errors.

#### rules

```php
rules(): array
```

Defines the specific validation criteria (e.g., `required`, `email`, `unique`) for each field.

- **Example**: `return ['email' => ['required' => true, 'type' => 'email']];`.

#### parameters

```php
parameters(): array
```

Maps internal field names to human-readable labels used in error messages.

- **Example**: `return ['user_email' => 'Email Address'];`.

---

### Logic & Flow Control

#### optional

```php
optional(): array
```

Triggers validation for secondary fields only if a primary "control" field matches a specific value.

- **Use Case**: Requiring "Company Name" only if "User Type" is set to "Business".

#### notempty

```php
notempty(): array
```

Defines fields that are not strictly required but must follow specific rules if the user provides any value.

- **Use Case**: An optional "Bio" field that must be at least 10 characters long if filled.

#### cleanup

```php
cleanup(array $requestData): array
```

Called at the very beginning of the validation cycle. Use it to strip out internal tokens or meta-fields.

- **Example**: `unset($requestData['_csrf_token']); return $requestData;`.

#### transformData

```php
transformData(array $data): array
```

Allows sanitizing or modifying data before rules are applied (e.g., trimming, lowercasing).

- **Example**: `$data['email'] = strtolower(trim($data['email']));`.

#### `modify(): array`

Renames fields in the final validated output.

```php
public function modify(): array
{
    return ['user_email' => 'email']; // 'user_email' becomes 'email' in validated() data
}
```

### File Handling

#### `file(): array`

Define labels for file upload fields. This ensures `$_FILES` data is correctly mapped and labeled.

```php
public function file(): array
{
    return ['avatar' => 'Profile Picture'];
}
```

#### Secure File Upload Validation

For comprehensive file upload security, use the `secure_file` validation rule:

```php
use Helpers\Http\FileHandler;

class UploadFormRequestValidation extends BaseRequestValidation
{
    public function expected(): array
    {
        return ['avatar', 'document'];
    }

    public function rules(): array
    {
        return [
            'avatar' => [
                'required' => true,
                'secure_file' => [
                    'type' => 'image',
                    'maxSize' => '2mb', // Human-readable format!
                ],
            ],
            'document' => [
                'secure_file' => [
                    'mimeTypes' => ['application/pdf'],
                    'extensions' => ['pdf'],
                    'maxSize' => '10mb', // Much clearer than 10485760
                ],
            ],
        ];
    }

    public function file(): array
    {
        return [
            'avatar' => 'Profile Picture',
            'document' => 'Document',
        ];
    }
}

// In your controller
public function upload(UploadFormRequestValidation $validator): Response
{
    $validator->validate([
        'avatar' => new FileHandler($_FILES['avatar']),
        'document' => new FileHandler($_FILES['document']),
    ]);

    if ($validator->has_error()) {
        return $this->response->status(400)->json($validator->errors());
    }

    // Files are validated - safe to upload
    $data = $validator->validated();
}
```

**`secure_file` Options:**

- `type`: Preset type (`'image'`, `'document'`, `'archive'`)
- `maxSize`: Maximum file size (supports human-readable formats or bytes)
- `mimeTypes`: Array of allowed MIME types (custom validation)
- `extensions`: Array of allowed extensions (custom validation)

#### File Size Formats

The `maxSize` option supports both numeric bytes and human-readable formats for better readability:

**Supported formats:**

- `'2mb'` or `'2MB'` - Megabytes
- `'500kb'` or `'500KB'` - Kilobytes
- `'1gb'` or `'1GB'` - Gigabytes
- `'1024b'` or `'1024B'` - Bytes
- `2097152` - Raw bytes (backward compatible)

**Examples:**

```php
'maxSize' => '2mb'      // 2 megabytes
'maxSize' => '500kb'    // 500 kilobytes
'maxSize' => '1gb'      // 1 gigabyte
'maxSize' => '1.5mb'    // 1.5 megabytes (decimals supported)
'maxSize' => 2097152    // 2MB in bytes (still works)
```

See [Security Documentation](security.md#file-uploads) for more details.

### Output Customization

#### `include(): array`

Specify fields to include in the `validated()` data object even if they were not in the input or were optional/empty.

```php
public function include(): array
{
    return ['referral_code'];
}
```

#### `exclude(): array`

Specify fields to exclude from the `validated()` data object, even if they were valid.

```php
public function exclude(): array
{
    return ['confirm_password', 'terms_agreed'];
}
```

#### `messages(): array`

Define custom error messages for specific fields and rules.

```php
public function messages(): array
{
    return [
        'email' => [
            'required' => 'We need your email address to sign you in.',
            'type' => 'Please provide a valid email address.',
        ],
    ];
}
```

#### `additionalData(): ?array`

Inject extra data into the validation process that wasn't in the form submission.

```php
public function additionalData(): ?array
{
    return ['user_id' => $this->auth->id()];
}
```

### Available Validation Rules Reference

Use these rules in your `rules()` array.

#### General & Type Rules

| Rule                 | Description                                                                                                                                                                 | Example                                   |
| :------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------- |
| `required`           | Field must be present and not empty.                                                                                                                                        | `'required' => true`                      |
| `type`               | Validates data type. Options: `email`, `password`, `phone`, `string`, `boolean`, `integer`, `numeric`, `url`, `alnum`, `alpha`, `coordinate`, `ipv4`, `ipv6`, `creditcard`. | `'type' => 'creditcard'`                  |
| `date`               | Validates date format.                                                                                                                                                      | `'date' => 'Y-m-d'`                       |
| `custom`             | Custom closure validation.                                                                                                                                                  | `'custom' => function($val) { ... }`      |
| `regex`              | Matches custom regex pattern.                                                                                                                                               | `'regex' => '/^[A-Za-z]+$/'`              |
| `is_valid`           | Matches preset regex (logic: `is`).                                                                                                                                         | `'is_valid' => 'alphanumeric'`            |
| `contains_valid`     | Checks if string contains pattern (logic: `has`).                                                                                                                           | `'contains_valid' => 'uppercase'`         |
| `contains_any_valid` | Checks if string contains ANY of patterns.                                                                                                                                  | `'contains_any_valid' => 'special_chars'` |

**Available `is_valid` Patterns:**
`int`, `float`, `email`, `url`, `zip`, `alpha`, `num`, `alphanum`, `ipv4`, `ipv6`, `username`, `password`, `creditcard`, `uuid`, `timestamp`, `iso8601`, `date`, `time`, `datetime`, `daterange`, `hexcolor`, `html`, `bbcode`, `intphone`, `address`, `fullname`, `name`, `lastname`.

#### String & Comparison Rules

| Rule           | Description                                 | Example                          |
| :------------- | :------------------------------------------ | :------------------------------- |
| `minlength`    | Minimum string length.                      | `'minlength' => 8`               |
| `maxlength`    | Maximum string length.                      | `'maxlength' => 255`             |
| `length`       | Exact string length (or set of lengths).    | `'length' => '10,12'`            |
| `limit`        | Numeric range check (min \| max).           | `'limit' => '18\|100'`           |
| `less_than`    | Must be less than another field's value.    | `'less_than' => 'max_budget'`    |
| `greater_than` | Must be greater than another field's value. | `'greater_than' => 'min_budget'` |

#### Field Matching Rules

| Rule          | Description                              | Example                        |
| :------------ | :--------------------------------------- | :----------------------------- |
| `same`        | Must allow match another field's value.  | `'same' => 'password'`         |
| `not_same`    | Must NOT match another field's value.    | `'not_same' => 'old_password'` |
| `confirm`     | Checks for `*_confirmation` field.       | `'confirm' => 'password'`      |
| `not_contain` | Must not contain chars from named field. | `'not_contain' => 'username'`  |

#### Database Rules

| Rule     | Description                                                                      | Example                                          |
| :------- | :------------------------------------------------------------------------------- | :----------------------------------------------- |
| `unique` | Fails if record ALREADY exists. Supports optional `:ignoreValue[,ignoreColumn]`. | `'unique' => 'user.email' . ($id ? ":$id" : '')` |
| `exist`  | Fails if record DOES NOT exist.                                                  | `'exist' => 'role.id'`                           |

### Unique Rule with Ignore ID

When updating a record, you often want to ensure a field (like email) remains unique, but you should exclude the current record's ID to prevent validation from failing against itself.

```php
namespace App\Account\Validations\Form;

use App\Core\BaseRequestValidation;

class UserUpdateValidation extends BaseRequestValidation
{
    public function expected(): array
    {
        return ['id', 'email', 'name'];
    }

    public function rules(): array
    {
        $id = $this->getRequestData()->get('id');

        return [
            'email' => [
                'type' => 'email',
                // Syntax: table.column:ignoreId
                // If $id is 5, it checks for unique email WHERE id != 5
                'unique' => 'user.email' . ($id ? ":$id" : ''),
            ],
            'name' => ['required' => true]
        ];
    }
}
```

### Robust Unique Rule (Non-ID Columns)

If your application uses UUIDs or slugs instead of auto-incrementing IDs, you can specify the ignore column as well.

**Syntax:** `table.column:ignoreValue,ignoreColumn`

```php
public function rules(): array
{
    $uuid = $this->getRequestData()->get('uuid');

    return [
        'slug' => [
            'type' => 'string',
            // Checks for unique slug WHERE uuid != $uuid
            'unique' => "post.slug:{$uuid},uuid",
        ],
    ];
}
```

#### File Validation Rules

| Rule                | Description                          | Example                                                  |
| :------------------ | :----------------------------------- | :------------------------------------------------------- |
| `file`              | Validates input is an uploaded file. | `'file' => true`                                         |
| `allowed_file_type` | Allowed file extensions.             | `'allowed_file_type' => ['jpg', 'png']`                  |
| `allowed_file_size` | Max file size.                       | `'allowed_file_size' => '2mb'`                           |
| `secure_file`       | Advanced file validation.            | See [Secure File Upload](#secure-file-upload-validation) |

#### Email Validation Rules

| Rule            | Description                    | Example                                 |
| :-------------- | :----------------------------- | :-------------------------------------- |
| `strict`        | Disposable/role/MX/SMTP check. | `'strict' => 'disposable,mx,smtp'`      |
| `allow_domains` | Whitelist specific domains.    | `'allow_domains' => '*.corp.com'`       |
| `block_domains` | Blacklist specific domains.    | `'block_domains' => 'spam.com'`         |
| `email_pattern` | Custom email regex.            | `'email_pattern' => '/@company\.com$/'` |

#### Rule Details

**Type Verification (`type`)**

- `email`: Validates standard email format.
- `phone`: Validates standard phone number format.
- `password`: Validates password strength. Configurable via `config` rule:
  ```php
  'password' => [
      'type' => 'password',
      'config' => [
          'uppercase' => 1,  // Min uppercase chars
          'lowercase' => 1,  // Min lowercase chars
          'special' => 1,    // Min special chars
          'numeric' => 1,    // Min numeric chars
          'length_min' => 8, // Min length
          'length_max' => 64,// Max length
          'not_common' => true, // Check common password list
          'not_in' => ['123', 'guest'] // Forbidden values
      ]
  ]
  ```
- `creditcard`: Validates credit card number (Luhn algorithm).
- `ipv4` / `ipv6`: Validates IP address formats.
- `coordinate`: Validates "lat,long" string.
- `url`: Validates URL format.
- `boolean`: Validates boolean value.
- `integer` / `numeric`: Validates numeric values.
- `alnum` / `alpha`: Validates alphanumeric / alphabetic only.

**Database Checks (`unique`, `exist`)**
Format: `'table_name.column_name'` or pass a **Closure**.

- **String Option:** `'unique' => 'user.email'` or `'unique' => 'user.email:5'` (ignores ID 5) or `'unique' => "user.email:{$uuid},uuid"` (ignores specific UUID).
- **Closure Option:** `'unique' => function($value) { return User::exists($value); }` (Custom logic)
- **Array Option (exist only):** `'exist' => ['male', 'female']` (Checks if value is in the allowed list)

- `unique`: Use for registration (ensure email is new).
- `exist`: Use for foreign keys (ensure role_id exists), login (ensure email exists), or whitelisting values.

**Comparison Rules (`limit`)**
Use pipe `|` to separate min and max.

- `'limit' => '10|'` (Min 10)
- `'limit' => '|100'` (Max 100)
- `'limit' => '10|100'` (Between 10 and 100)

**Regex Presets (`is_valid`, `contains_valid`)**
Common patterns supported by the framework's `ValidationTrait`:

- `alphanumeric`: Letters and numbers.
- `uppercase`: Contains uppercase letters.
- `lowercase`: Contains lowercase letters.
- `special_chars`: Contains special characters.

**Custom Validation (`custom`)**
Pass a Closure or callable that returns `true` (pass) or `false` (fail).

```php
'custom' => function($value) {
    return $value === 'secret';
}
```

### Accessing Results

These methods are used in your controller after calling `validate()`.

| Method                               | Description                                                                      |
| :----------------------------------- | :------------------------------------------------------------------------------- |
| `validate(array $requestData): void` | Executes the validation process.                                                 |
| `has_error(): bool`                  | Returns `true` if validation failed.                                             |
| `errors(): array`                    | Returns an associative array of validation errors.                               |
| `validated(): ?Data`                 | Returns the validated, cleaned, and transformed data as a `Helpers\Data` object. |
