# Security

Anchor provides comprehensive security features to protect your application against common web vulnerabilities. This guide covers all security features and best practices.

## Security Features Overview

| Feature                      | Protection Against          | Documentation                             |
| ---------------------------- | --------------------------- | ----------------------------------------- |
| **CSRF Protection**          | Cross-Site Request Forgery  | [csrf](csrf.md)                           |
| **SQL Injection Protection** | SQL Injection               | [Query Builder](query-builder.md)         |
| **XSS Protection**           | Cross-Site Scripting        | [views](views.md)                         |
| **Authentication**           | Unauthorized Access         | [authentication](authentication.md)       |
| **Encryption**               | Data Breaches               | [encryption](encryption.md)               |
| **Firewall**                 | Brute Force, Rate Limiting  | [firewall](firewall.md)                   |
| **Security Headers**         | Clickjacking, MIME Sniffing | [middleware](middleware#security-headers) |
| **File Upload Validation**   | Malicious Uploads           | [#file-uploads](#file-uploads)            |
| **Input Validation**         | Invalid Data                | [validation](validation.md)               |

## Quick Security Checklist

### Production Deployment

- `APP_KEY` generated and secured
- HTTPS enabled (`APP_SECURE=true`)
- Debug mode disabled (`APP_DEBUG=false`)
- CSRF protection enabled
- Security headers middleware enabled
- Strong password policies enforced
- File upload validation implemented
- Database credentials secured
- Error reporting configured properly
- Session timeout configured
- Firewall thresholds set appropriately

### Code Security

- All user input validated
- Output escaped in views (`$this->escape()`)
- Parameterized queries used (Query Builder)
- Passwords hashed with Argon2ID
- Sensitive data encrypted at rest
- Authorization checks implemented
- File uploads validated
- CORS configured for APIs (if needed)

## CSRF Protection

**Enabled by default** - Protects against Cross-Site Request Forgery attacks.

```php
// In forms
<form method="POST">
    <?php echo $this->csrf(); ?>
    <!-- form fields -->
</form>

// In AJAX
fetch('/api/endpoint', {
    headers: {
        'X-CSRF-Token': '<?= csrf_token() ?>'
    }
});
```

**Configuration:** `App/Config/default.php` → `csrf`

See [csrf](csrf.md) for details.

## SQL Injection Protection

**Built-in** - Query Builder uses parameterized queries.

```php
// Safe - uses bindings
User::query()->where('email', '=', $email)->first();

// Safe - uses bindings
DB::table('user')->where('id', '=', $id)->get();

// Use whereRaw() carefully
DB::table('user')->whereRaw('created_at > NOW()')->get();
```

## XSS Protection

**Automatic escaping** in views + **Security Headers** middleware.

```php
// Safe - automatically escaped
<?= $this->escape($user->name) ?>

// Unsafe - only use for trusted HTML
<?= $trustedHtml ?>
```

**Security Headers:**

- `X-XSS-Protection: 1; mode=block`
- `X-Content-Type-Options: nosniff`
- `Content-Security-Policy` (optional)

## Authentication & Authorization

**Argon2ID password hashing** + **Session-based authentication**.

```php
// Hash password
$hashedPassword = enc()->hashPassword('user-password');

// Verify password
if (enc()->verifyPassword('input', $hashedPassword)) {
    // Valid
}

// Check authentication
if ($this->auth->isAuthenticated()) {
    // User is logged in
}

// Check authorization
if ($this->auth->isAuthorized($route)) {
    // User can access this route
}
```

See [authentication.md](authentication.md) for details.

## Encryption

**AES-256-GCM** for strings, **Argon2ID** for passwords, **PBKDF2** for files.

```php
// Encrypt sensitive data
$encrypted = encrypt('sensitive data');

// Decrypt
$decrypted = decrypt($encrypted);

// Hash password
$hash = enc()->hashPassword('password');

// Verify password
if (enc()->verifyPassword('password', $hash)) {
    // Valid
}
```

See [encryption.md](encryption.md) for details.

## Security Headers

**Middleware** that adds security headers to all responses.

```php
// In App/Config/middleware.php
return [
    'web' => [
        \App\Middleware\Web\SecurityHeadersMiddleware::class,
        // ... other middleware
    ],
];
```

**Headers Added:**

- `X-Frame-Options: SAMEORIGIN` - Prevents clickjacking
- `X-Content-Type-Options: nosniff` - Prevents MIME sniffing
- `X-XSS-Protection: 1; mode=block` - Legacy XSS protection
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy` - Controls browser features
- `Strict-Transport-Security` - HSTS (HTTPS only)

**Configuration:** `App/Config/default.php` → `security_headers`

```php
'security_headers' => [
    'enabled' => true,
    'x_frame_options' => 'SAMEORIGIN',
    'x_content_type_options' => 'nosniff',
    'hsts_enabled' => true,
];
```

## Firewall and Rate Limiting

**Protects against brute force** and excessive requests.

```php
use Security\Firewall\Drivers\AccountFirewall;

$firewall = resolve(AccountFirewall::class);

$firewall->user(['id' => $userId])
    ->handle();

if ($firewall->isBlocked()) {
    // User is blocked
}
```

See [firewall](firewall.md) for details.

## File Uploads

**Securely handle file uploads** to prevent malicious code execution.

```php
$file = $this->request->file('avatar');

// Validate type, size, and use a safe filename
$path = $file->moveSecurely('/uploads/avatars', [
    'type' => 'image',       // Only allow images
    'maxSize' => 2097152,    // Max 2MB
    'extensions' => ['jpg', 'png'] // Explicit extensions
]);

if (!$path) {
    // Validation failed
    $error = $file->getValidationError();
}
```

**Key Security Features:**

- **MIME Type Validation**: Checks actual file content, not just extension
- **Extension Allow-list**: Only allows safe file extensions
- **Filename Sanitization**: Generates random safe filenames or sanitizes input
- **Size Limits**: Prevents DoS attacks via large files
- **Permissions**: Sets non-executable permissions on uploaded files

See [requests.md#file-uploads](requests.md#file-uploads) for implementation details.

## CORS (Cross-Origin Resource Sharing)

**For APIs** - Configure allowed origins, methods, and headers.

```php
// Enable in App/Config/cors.php
'enabled' => true,
'allowed_origins' => [
    'https://yourdomain.com',
    'https://*.yourdomain.com', // Wildcard subdomain
],
'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE'],
```

**Add to API middleware:**

```php
// In App/Config/middleware.php
return [
    'api' => [
        \App\Middleware\Api\CorsMiddleware::class,
        // ... other middleware
    ],
];
```

## Input Validation

**Class-based validation** with type checking and sanitization.

```php
public function rules(): array
{
    return [
        'email' => ['type' => 'email'],
        'password' => [
            'type' => 'password',
            'config' => [
                'uppercase' => 1,
                'numeric' => 1,
                'special' => 1,
                'length_min' => 12, // Increased for security
                'not_common' => true,
            ]
        ],
    ];
}
```

See [validation](validation.md) for details.

## Session Security

**Database-backed sessions** with secure cookies.

```php
// Configuration in App/Config/default.php
'session' => [
    'timeout' => 14400, // 4 hours
    'cookie' => [
        'secure' => true,      // HTTPS only
        'http_only' => true,   // No JavaScript access
        'samesite' => 'Lax',   // CSRF protection
    ],
],
```

## Security Best Practices

### Use HTTPS in Production

```ini
# .env
APP_SECURE=true
```

### Validate All User Input

```php
// Always validate before processing
$validator->validate($this->request->post());

if ($validator->has_error()) {
    // Handle errors
}
```

### Escape All Output

```php
// In views
<?= $this->escape($user->name) ?>
```

### Use Parameterized Queries

```php
// Safe
User::query()->where('email', '=', $email)->first();

// Never do this
DB::raw("SELECT * FROM user WHERE email = '$email'");
```

### Hash Passwords

**Don't Encrypt**

```php
// Correct
$user->password = enc()->hashPassword($password);

// Wrong
$user->password = encrypt($password);
```

### Protect Sensitive Data

```php
// Encrypt sensitive data at rest
$user->ssn = encrypt($ssn);

// Use HTTPS for data in transit
```

### Implement Authorization

```php
// Check if user can access resource
if (!$this->auth->isAuthorized($route)) {
    return $this->response->status(403)->json(['error' => 'Forbidden']);
}
```

### Keep Dependencies Updated

```bash
composer update
```

### Use Security Headers

Enable `SecurityHeadersMiddleware` in production.

### Regular Security Audits

- Review code for vulnerabilities
- Update dependencies
- Test authentication/authorization
- Verify HTTPS configuration
- Check file upload handling

## Common Vulnerabilities & Protections

| Vulnerability                | Protection             | Status        |
| ---------------------------- | ---------------------- | ------------- |
| **SQL Injection**            | Parameterized queries  | ✅ Built-in   |
| **XSS**                      | Output escaping + CSP  | ✅ Built-in   |
| **CSRF**                     | Token validation       | ✅ Built-in   |
| **Clickjacking**             | X-Frame-Options        | ✅ Middleware |
| **MIME Sniffing**            | X-Content-Type-Options | ✅ Middleware |
| **Session Fixation**         | Session regeneration   | ✅ Built-in   |
| **Brute Force**              | Firewall rate limiting | ✅ Built-in   |
| **Weak Passwords**           | Argon2ID hashing       | ✅ Built-in   |
| **Insecure Deserialization** | Class whitelists       | ✅ Built-in   |
| **Malicious Uploads**        | File validation        | ✅ Utility    |

## Compliance

The framework's security features support compliance with:

- **PCI DSS** - Payment card data protection
- **HIPAA** - Healthcare data encryption
- **GDPR** - Personal data protection
- **SOC 2** - Security controls

## Reporting Security Issues

If you discover a security vulnerability, please email security@example.com. Do not create public issues for security vulnerabilities.

## Additional Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CSRF Documentation](csrf.md)
- [Encryption Documentation](encryption.md)
- [Authentication Documentation](authentication.md)
- [Firewall Documentation](firewall.md)
- [Validation Documentation](validation.md)
