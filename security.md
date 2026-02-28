# Security

Anchor provides comprehensive security features to protect your application against common web vulnerabilities. This guide covers the frameworks approach to securing data, identities, and resources.

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
| **Open Redirects**           | Phishing, Malicious Links   | [#open-redirects](#open-redirects)        |
| **File Upload Validation**   | Malicious Uploads           | [#file-uploads](#file-uploads)            |

## Quick Security Checklist

### Production Deployment

- `APP_KEY` generated and secured (`php dock make:key`)
- HTTPS enabled (`APP_SECURE=true`)
- Debug mode disabled (`APP_DEBUG=false`)
- CSRF protection enabled in `App/Config/default.php`
- Firewall thresholds set appropriately in `App/Config/firewall.php` (Automatically integrated)
- Security headers middleware enabled (Add to `App/Config/middleware.php` if not using system defaults)

### Code Security

- All user input validated via Request classes
- Output escaped in views using `<?= $this->escape($var) ?>`
- Parameterized queries used via the Query Builder
- Passwords hashed with Argon2ID via `enc()->hashPassword()`
- Sensitive data encrypted at rest using `encrypt()`
- Authorization checks implemented via `$this->auth->isAuthorized()`

## CSRF Protection

Anchor protects against Cross-Site Request Forgery by default for all state-changing requests (POST, PUT, DELETE). It uses **constant-time comparisons** (`hash_equals`) for both the token and the origin-check hash to prevent timing attacks.

```php
// In forms
<form method="POST">
    <?php echo $this->csrf(); ?>
    <!-- form fields -->
</form>

// In AJAX (using the global helper)
fetch('/api/endpoint', {
    headers: {
        'X-CSRF-Token': '<?= csrf_token() ?>'
    }
});
```

**Configuration:** `App/Config/default.php` → `csrf`


## SQL Injection Protection

The framework core utilizes PDO with parameterized queries for all database operations, making SQL injection nearly impossible when using standard methods.

```php
// Safe - uses bindings automatically
User::query()->where('email', '=', $email)->first();

// Safe - manual bindings for complex queries
DB::query("SELECT * FROM users WHERE status = ?", ['active'])->get();
```

## XSS Protection

Anchor provides multi-layered XSS protection:
- **Output Escaping**: The view system provides `$this->escape()` for safe output.
- **Security Headers**: Middleware adds `X-XSS-Protection` and `X-Content-Type-Options` headers.

```php
// Safe - automatically escaped if using the escape method
<?= $this->escape($user->bio) ?>

// Unsafe - use ONLY for trusted content
<?= $user->html_content ?>
```

## Authentication & Authorization

Anchor uses **Argon2ID** for password hashing, which is the current industry standard.

### Password Management

Use the `enc()` helper to handle password operations securely:

```php
// Hash a password using Argon2ID
$hashedPassword = enc()->hashPassword('secure-password');

// Verify a password
if (enc()->verifyPassword('input-password', $hashedPassword)) {
    // Valid
}
```

### Access Control

Verify a user's status and permissions through the `AuthManager`:

```php
// Check if user is logged in
if ($this->auth->isAuthenticated()) { ... }

// Check route-level authorization
if ($this->auth->isAuthorized($route)) { ... }
```

## Encryption

Anchor provides high-level drivers for data and file encryption:

- **Strings**: AES-256-GCM (via `encrypt()` / `decrypt()`)
- **Passwords**: Argon2ID (via `enc()->hashPassword()`)
- **Files**: PBKDF2 with high iteration counts.

```php
// Encrypt sensitive string data
$encrypted = encrypt('personal-info');

// Decrypt
$decrypted = decrypt($encrypted);
```

## Firewall and Rate Limiting

The Firewall system protects against brute force attacks and malicious traffic.

### Automated Integration

The firewall is automatically integrated into the authentication flow via system events:

- **`LoginEvent`**: Resets the failure counter upon success (`LoginListener`).
- **`LoginFailedEvent`**: Increments the failure counter and triggers blocks (`LoginFailedListener`).

### Manual Usage

You can also trigger firewall checks manually in your services or controllers:

```php
use Security\Firewall\Drivers\AccountFirewall;

$firewall = resolve(AccountFirewall::class);

$firewall->user(['id' => $userId])->handle();

if ($firewall->isBlocked()) {
    // Return error or redirect
}
```

## File Uploads

Never trust user-supplied filenames or MIME types. Use the `moveSecurely()` method on uploaded files.

```php
$path = $file->moveSecurely('/uploads/avatars', [
    'type' => 'image',             // Validation against actual file content
    'maxSize' => 2 * 1024 * 1024,  // 2MB
    'extensions' => ['jpg', 'png'] // Only allow these extensions
]);
```

## Open Redirect Protection

The framework prevents open redirect vulnerabilities by validating redirect URLs. By default, `Response::redirect()` only allows URLs internal to your application.

```php
// Safe - internal URL
return $this->response->redirect('/dashboard');

// Blocked by default - external URL
return $this->response->redirect('https://malicious.com');

// Explicitly allow external redirect
return $this->response->redirect('https://trusted-site.com', 302, true);
```

## Path Traversal Protection

Anchor provides utilities to ensure file operations stay within allowed directory trees, preventing `../` traversal attacks.

```php
use Helpers\File\Paths;

// Resolves path and throws exception if it traverses outside the project root
$safePath = Paths::securePath($userInputPath);

// Verify if a path is within a specific base
if (FileSystem::isWithin($path, storage_path())) {
    // Authorized access
}
```

## Session Security

Sessions are secured using several techniques:

- **Database Storage**: Prevents session hijacking via local file access.
- **Secure Cookies**: `http_only`, `secure`, and `SameSite=Lax` are defaults.
- **Rotation**: `SessionGuard` automatically regenerates the session ID upon login and logout to prevent fixation attacks.

## Common Vulnerabilities & Protections

| Vulnerability                | Protection             | Status        |
| ---------------------------- | ---------------------- | ------------- |
| **SQL Injection**            | Parameterized queries  | ✅ Built-in   |
| **XSS**                      | Output escaping + CSP  | ✅ Built-in   |
| **CSRF**                     | Token validation (constant-time) | ✅ Built-in   |
| **Open Redirects**           | Internal URL validation          | ✅ Built-in   |
| **Clickjacking**             | X-Frame-Options                  | ✅ Middleware |
| **MIME Sniffing**            | X-Content-Type-Options           | ✅ Middleware |
| **Session Fixation**         | Session regeneration   | ✅ Built-in   |
| **Brute Force**              | Firewall rate limiting | ✅ Built-in   |
| **Weak Passwords**           | Argon2ID hashing       | ✅ Built-in   |
| **Insecure Deserialization** | Class whitelists       | ✅ Built-in   |
| **Malicious Uploads**        | File validation        | ✅ Utility    |

## Compliance

Anchor's security architecture supports compliance standards including **GDPR**, **PCI DSS**, and **HIPAA** through its implementation of encryption at rest, secure data handling, and comprehensive auditing.
