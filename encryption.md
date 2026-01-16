# Encryption

Anchor provides robust encryption services using industry-standard algorithms. The framework supports both string encryption (AES-256-GCM) and file encryption (AES-256-GCM with PBKDF2 key derivation), as well as secure password hashing (Argon2ID).

## Configuration

Set your `APP_KEY` in `.env`. This must be a base64-encoded 32-byte (256-bit) key.

```ini
APP_KEY=base64:your-32-byte-key-here-encoded-in-base64
```

> Use the CLI to generate a secure key automatically:

```bash
php dock key:generate
```

## String Encryption

### Basic Usage (Global Helpers)

The simplest way to encrypt and decrypt data is using the global helper functions.

```php
// Encrypt string
$encrypted = encrypt('sensitive data');

// Decrypt string
$decrypted = decrypt($encrypted);
```

### Encrypting Arrays

The `encrypt()` helper automatically serializes and encrypts arrays, preserving their structure upon decryption.

```php
$data = [
    'credit_card' => '4111111111111111',
    'cvv' => '123'
];

$encrypted = encrypt($data);
// Value is now a single encrypted string

$decrypted = decrypt($encrypted);
// Returns original array ['credit_card' => ..., 'cvv' => ...]
```

### Advanced Usage (Encrypter Class)

For explicit control or when using dependency injection, use the `Encrypter` class via the `enc()` helper or the container.

```php
// Access via helper
$encrypter = enc();

// Explicitly use string driver
$encrypted = $encrypter->string()->encrypt('data');
$decrypted = $encrypter->string()->decrypt($encrypted);
```

## Password Hashing

Anchor uses **Argon2ID**, the winner of the Password Hashing Competition, offering superior resistance against GPU cracking attacks compared to Bcrypt.

> Never use reversible encryption (`encrypt()`) for passwords. Always use one-way hashing (`enc()->hashPassword()`).

### Hashing & Verifying

```php
// 1. Hash a password
$hashedPassword = enc()->hashPassword('user-input-password');

// 2. Verify a password
if (enc()->verifyPassword('user-input-password', $storedHash)) {
    // Password is correct
}
```

## File Encryption

Encrypt and decrypt entire files using password-based encryption. This uses PBKDF2 for secure key derivation, making it safe for portable documents.

```php
// Encrypt a file
enc()->file()
    ->password('strong-secret-password')
    ->encrypt(
        source: storage_path('docs/contract.pdf'),
        destination: storage_path('vault/contract.enc')
    );

// Decrypt a file
$content = enc()->file()
    ->password('strong-secret-password')
    ->decrypt(storage_path('vault/contract.enc'));
```

## Use Case

### Encrypting Model Attributes

Automatically encrypt sensitive data in your models.

```php
class User extends BaseModel
{
    // Mutator: Encrypt on set
    public function setSsnAttribute($value)
    {
        $this->attributes['ssn'] = encrypt($value);
    }

    // Accessor: Decrypt on get
    public function getSsnAttribute()
    {
        return decrypt($this->attributes['ssn']);
    }
}
```

### Storing API Keys

Securely store third-party credentials in your database.

```php
// Storing
DB::table('api_keys')->insert([
    'service' => 'stripe',
    'key' => encrypt('sk_live_...'),
]);

// Retrieving
$record = DB::table('api_keys')->where('service', 'stripe')->first();
$apiKey = decrypt($record['key']);
```

### Secure File Uploads

Encrypt user uploads immediately upon receipt.

```php
public function upload(array $file)
{
    $safeName = uniqid() . '.enc';

    enc()->file()
        ->password(env('FILE_ENCRYPTION_KEY'))
        ->encrypt($file['tmp_name'], storage_path('uploads/' . $safeName));

    return $safeName;
}
```

## Technical Specifications

| Feature               | Algorithm            | Specs                                                                       |
| :-------------------- | :------------------- | :-------------------------------------------------------------------------- |
| **String Encryption** | AES-256-GCM          | 256-bit key, Random IV (12 bytes), Auth Tag (16 bytes)                      |
| **File Encryption**   | AES-256-GCM + PBKDF2 | SHA-256 hash, 65,536 iterations, Random Salt (16 bytes)                     |
| **Password Hashing**  | Argon2ID             | Memory-hard, time-cost configurable, hybrid generic/side-channel protection |

**Security Guarantees:**

- **Confidentiality**: Data cannot be read without the key.
- **Integrity**: Encrypted data includes an authentication tag (GCM mode). Any tampering triggers a decryption error.
- **Uniqueness**: Random IVs ensure the same plaintext produces different ciphertext every time.

## Security Best Practices

1.  **Protect Your APP_KEY**: If this key is leaked, all encrypted data is compromised. Never commit it to version control.
2.  **Rotate Keys Gracefully**

    Anchor supports zero-downtime key rotation. To rotate your app key:

    - Generate a new key: `php dock key:generate --show`.
    - Add your _current_ key to `APP_PREVIOUS_KEYS` in `.env` (comma-separated).
    - Update `APP_KEY` with the _new_ key.

```ini
APP_KEY=base64:NEW_KEY...
APP_PREVIOUS_KEYS=base64:OLD_KEY_1...,base64:OLD_KEY_2...
```

Anchor will automatically:

- Encrypt new data with the new `APP_KEY`.
- Decrypt old data using keys from `APP_PREVIOUS_KEYS` if the primary key fails.
- (Optional) You can then run a script to re-encrypt data to fully migrate to the new key.

3.  **Use HTTPS**: Encryption protects data at rest; HTTPS protects data in transit.
4.  **Handle Errors**: Decryption throws exceptions if data is tampered with. Always use try-catch blocks in critical flows.

## Troubleshooting

### Encryption

Encryption key must be 32 bytes

Your `APP_KEY` in `.env` is invalid or missing. Run `php dock key:generate`.

### Decryption

Decryption failed or data is corrupt

This `RuntimeException` means:

- The data was modified (tampered) after encryption.
- The `APP_KEY` has changed since encryption.
- You are trying to decrypt data with the wrong password (for file encryption).

### Password hashing failed

Ensure your PHP installation is compiled with Sodium or Argon2 support.

## Method Reference

| Helper / Method                     | Description                                |
| :---------------------------------- | :----------------------------------------- |
| `encrypt($value)`                   | Global helper for string/array encryption. |
| `decrypt($payload)`                 | Global helper for string/array decryption. |
| `enc()`                             | Returns the `Encrypter` instance.          |
| `enc()->hashPassword($str)`         | Hashes a password using Argon2ID.          |
| `enc()->verifyPassword($pt, $hash)` | Verifies a password against a hash.        |
| `enc()->file()->encrypt(...)`       | Encrypts a file on disk.                   |
| `enc()->file()->decrypt(...)`       | Decrypts a file on disk.                   |
