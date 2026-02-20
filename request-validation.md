# Request Validation

Anchor provides a robust, class-based approach to request validation for both web forms and API requests. By creating dedicated validation classes that extend `App\Core\BaseRequestValidation`, you can encapsulate validation logic, keep controllers clean, and easily reuse validation rules across different request types.

### Via CLI (Recommended)

The fastest way to create a validation class is using the `dock` CLI. **By default, this will also create the associated Request DTO.**

```bash
php dock validation:create {Name} {Module} --type={form|api} [--no-dto]
```

**Examples:**

```bash
# Atomic: Creates LoginFormRequestValidation AND LoginRequest DTO
php dock validation:create Login Auth --type=form

# Only creates the validator
php dock validation:create User --type=api --no-dto
```

**Cleanup:**
When deleting a validator via `php dock validation:delete`, the CLI will automatically offer to clean up the associated Request DTO to keep your codebase clean. You can also use the `--with-dto` flag to automate this.

### Manual Creation

To create a validation class manually, extend the `App\Core\BaseRequestValidation` abstract class. You are required to implement the `expected()` method, and optionally override `rules()` and `parameters()` methods.

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

Anchor promotes a **Zero Boilerplate** approach. When using Smart Validation, there is no need to manually instantiate or call the validator within your controller. The middleware handles execution and populates the request with the validated Data Transfer Object (DTO).

### Zero Boilerplate Pattern (Recommended)

```php
namespace App\Auth\Controllers;

use App\Core\BaseController;
use Helpers\Http\Response;

class LoginController extends BaseController
{
    public function attempt(): Response
    {
        // 1. Smart Validation has already run before this method.
        // 2. Retrieve the validated DTO directly from the request.
        $dto = $this->request->validated();

        // 3. Proceed with business logic
        if (! $this->auth->login($dto)) {
            return $this->response->redirect($this->request->fullRoute());
        }

        return $this->handleLoginRedirect();
    }
}
```

### Manual Injection (Legacy/Explicit)

If you prefer explicit types or need to perform manual validation steps, you can still inject the validation class:

```php
public function attempt(LoginFormRequestValidation $validator): Response
{
    // Run validation manually against specific data
    $validator->validate($this->request->post());

    if ($validator->has_error()) {
         return $this->handleFailure($validator->errors());
    }

    $dto = $validator->getRequest(); // Get the DTO from the validator
}
```

### Smart Validation

Anchor provides **Smart Validation**, a zero-config automation feature that uses [Route Context](route-context.md) to automatically resolve, execute, and handle validation results before your controller action even begins.

#### How it Works

The internal `Core\Middleware\SmartValidationMiddleware` automatically hooks into every request and:

- **Extracts Context**: Identifies the `domain`, `entity`, and `action` from the route.
- **Resolves Validation Class**: Searches for a matching validation class in `App\{Domain}\Validations\{Form|Api}` (or uses an explicit override).
- **Executes Validation**: Instantiates the class and runs `validate($request->all())`.

- **Handles Failure**:
  - **Web**: Automatically flashes input/errors and redirects back.
  - **API/AJAX**: Returns a structured `422 Unprocessable Entity` JSON response for consistency.

#### Explicit Validator Overrides

If the automated naming convention doesn't fit your use case, you can explicitly specify a validator class in the [Route Context](route-context.md). This is typically done in a custom middleware to keep your controllers clean:

```php
$request->setRouteContext('validator', UserFormRequestValidation::class);
```

When this key is present, the middleware skips automated resolution and uses the provided class.

#### AJAX Support

The middleware is AJAX-aware. If a request is made via `XMLHttpRequest` (or specifies `Accept: application/json`), validation failures will automatically result in a JSON response with a **422** status code, even if not on an API route.

#### Class Naming Convention

To enable Smart Validation, name your validation classes following these patterns:

- `App\Account\Validations\Form\UserFormRequestValidation` (Default)
- `App\Account\Validations\Api\UserApiRequestValidation` (API specific)
- `App\Account\Validations\Form\EditUserFormRequestValidation` (Action specific)

#### Benefits

- **Zero Boilerplate**: No need to inject or call validators manually.
- **Clean Controllers**: Controller methods stay focused on the "happy path" without validation logic.
- **Consistent Responses**: Standardized error handling for both Web (302 redirects) and API/AJAX (422 JSON).
- **Type-Safe Data**: Access already-validated DTOs via `$this->request->validated()`.

## Manual Validation

While Smart Validation handles most cases, you may occasionally need to trigger validation manually within a controller while still benefiting from the framework's failure handling logic.

### validateUsing

The `validateUsing()` method allows you to execute a specific validation class and automatically trigger the middleware's failure handler (e.g., redirecting with errors or returning JSON).

```php
public function store(): Response
{
    // Triggers failure logic (redirect/JSON) if validation fails
    $this->request->validateUsing(CustomValidator::class);

    // If we reach here, validation passed
    $data = $this->request->validated();

    $redirect_to = $this->request->fullRouteByName('home');

    if (! $this->request->isLoginRoute()) {
        $redirect_to = $this->request->callback();
    }

    return $this->response->redirect($redirect_to);
}
```

This is particularly useful when you need to perform conditional validation or multi-step logic before a validator is chosen.

## Use Cases

### The Multi-Step Wizard (Manual Trigger)

In a multi-step registration process, you might want to validate different parts of the request data at different stages, but still want the framework to handle the failure logic automatically.

```php
namespace App\Account\Controllers;

use App\Core\BaseController;
use Helpers\Http\Request;
use Helpers\Http\Response;
use App\Account\Validations\Form\AccountBasicValidation;
use App\Account\Validations\Form\AccountProfileValidation;

class RegistrationController extends BaseController
{
    public function store(): Response
    {
        $step = $this->request->get('step', 1);

        if ($step === 1) {
           // Triggers failure logic (redirect/JSON) if basic info is invalid
           $this->request->validateUsing(AccountBasicValidation::class);
        } else {
           // Triggers failure logic for step 2
           $this->request->validateUsing(AccountProfileValidation::class);
        }

        // If we reach here, validation passed
        $data = $this->request->validated();
        
        // Process data...
        return $this->response->redirect(route('next-step'));
    }
}
```

### Custom Admin Endpoints (Context Override)

Sometimes you have a standard `UserController` for public profile updates, but a separate `AdminUserController` that requires stricter validation (like role assignment) for the same `User` entity. **Instead of overriding the controller constructor**, the idiomatic Anchor pattern is to use a simple middleware to set the context.

```php
namespace App\Account\Middleware;

use Closure;
use Helpers\Http\Request;
use Helpers\Http\Response;
use App\Account\Validations\Form\AdminUserUpdateValidation;

class UseAdminValidatorMiddleware
{
    public function handle(Request $request, Response $response, Closure $next): Response
    {
        // Explicitly override the smart validator for this route group or controller
        $request->setRouteContext('validator', AdminUserUpdateValidation::class);
        
        return $next($request, $response);
    }
}
```

By applying this middleware to your admin routes, you bypass the default `UserFormRequestValidation` and use your specialized admin validator instead, without adding any boilerplate to your controller methods.

### SPA Interaction (AJAX Support)

If you are building a SPA (Vue, React, HTMX) and want to use the same controller action for both standard form submissions and AJAX requests, the `SmartValidationMiddleware` handles it automatically.

```php
// Controller logic stays clean and focused on the happy path
public function store(UserRequest $dto): Response
{
    User::create($dto->all());
    
    return $this->response->redirect('/users');
}
```

If an HTMX or fetch request fails validation, the middleware detects the `X-Requested-With` header or `Accept: application/json` and returns a **422 JSON** error object instead of a 302 redirect. This allows your frontend to display inline errors without extra backend logic.

## Validation Exceptions

When validation fails manually or via middleware, the framework throws an `Exceptions\ValidationException`. 

```php
use Exceptions\ValidationException;

try {
    // ... validation logic
} catch (ValidationException $e) {
    $errors = $e->getErrors();
}
```

## Email Validation

Anchor provides comprehensive email validation with **zero external dependencies**. Protect your application from disposable emails, role-based accounts, and enforce domain policies.

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

- Email format validation (`type => 'email'`)
- Strict checks (disposable, role, mx)
- Domain whitelist (`allow_domains`)
- Domain blacklist (`block_domains`)
- Pattern matching (`email_pattern`)

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

> Use `$request->validateUsing(UploadFormRequestValidation::class)` for even cleaner controller code when manual triggers are required.

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
