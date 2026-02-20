# Configuration

Anchor uses a combination of `.env` files and PHP configuration files to manage application settings.

## Environment Variables

The `.env` file at the root of your project contains sensitive and environment-specific variables.

**Important**: This file should **never** be committed to version control. Add it to `.gitignore`.

#### Example

```ini
# Application
APP_NAME=AnchorApp
APP_ENV=local
APP_DEBUG=true
APP_KEY=base64:your-32-character-key-here
APP_URL=localhost:1010

# Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_DATABASE=anchor_db
DB_USERNAME=root
DB_PASSWORD=secret

# Or SQLite
DB_CONNECTION=sqlite
DB_DATABASE=anchor.sqlite

# Mail
MAIL_DRIVER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=your_username
MAIL_PASSWORD=your_password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="${APP_NAME}"

# Timezone
TIMEZONE=UTC

# Session
SESSION_TIMEOUT=3600

# Slow Query Threshold (milliseconds)
SLOW_QUERY_THRESHOLD_MS=500
```

## Environment Functions

#### env

```php
env(string $key, mixed $default = null): mixed
```

Retrieves a variable from the `.env` file. It automatically handles type conversion for booleans (`true`/`false`), integers, and comma-separated lists (returning them as arrays).

- **Use Case**: Accessing sensitive secrets or environment-specific toggles (e.g., API keys, debug mode).
- **Example**: `$debug = env('APP_DEBUG', false);`.

## Securing Environment Files

In production environments, you may want to encrypt your `.env` file to safely include it in version control (or just for extra security). Use the `env:encrypt` and `env:decrypt` commands.

### Encrypting .env

Run the following command to encrypt your `.env` file to `.env.encrypted`:

```bash
php dock env:encrypt
```

You can optionally provide a specific key:

```bash
php dock env:encrypt --key=base64:your-key-here
```

### Decrypting .env

To restore your `.env` file from the encrypted version:

```bash
php dock env:decrypt --key=base64:your-key-here
```

### Runtime Decryption

Anchor supports "Zero-Disk" security by decrypting your `.env.encrypted` file directly in memory at runtime. This avoids having a plain `.env` file anywhere on your server.

To use runtime decryption:

- Ensure your `.env.encrypted` is present in the root directory.
- Ensure you have NO `.env` file present (or it will take precedence).
- Set your decryption key as a system environment variable: `ANCHOR_ENV_ENCRYPTION_KEY`.

Anchor will automatically detect the encrypted file and use the key to load your variables.

#### CI/CD Integration

You can set the `ANCHOR_ENV_ENCRYPTION_KEY` environment variable in your CI/CD pipeline. The `env:decrypt` command will automatically use it if no `--key` is provided.

#### Security

The encrypted file is secured with **AES-256-GCM**. Never lose your encryption key, as the data cannot be recovered without it.

## Configuration Files

Configuration files are located in `App/Config/` and return arrays of settings.

### Available Configuration Files

- **api.php** - API-specific settings
- **app.php** - Application branding and assets
- **auth.php** - Authentication guards and user sources
- **cache.php** - Cache drivers and settings
- **command.php** - CLI command settings
- **cors.php** - Cross-Origin Resource Sharing settings
- **database.php** - Database connections and settings
- **default.php** - Core settings (debug, env, session, csrf, auth)
- **email_validation.php** - Email validation rules
- **filesystems.php** - Filesystem disks and drivers
- **firewall.php** - Security and firewall rules
- **functions.php** - Custom functions
- **image.php** - Image processing settings
- **mail.php** - Email configuration (SMTP, drivers)
- **middleware.php** - Global and route middleware registry
- **permissions.php** - System permission registry
- **playground.php** - REPL/Playground configuration
- **providers.php** - Service provider registration
- **route.php** - Routes, groups, and URL settings

## Configuration Functions

### config

```php
config(string $key, mixed $default = null): mixed
```

Retrieves a configuration value using dot notation. If the first segment doesn't match a filename in `App/Config`, it searches `App/Config/default.php`.

- **Use Case**: Accessing structured, logic-based settings that are shared across the application.
- **Example**: `$timeout = config('session.timeout', 3600);`.

### Dot Notation

Configuration values are accessed using dot notation. If the first segment matches a filename in `App/Config`, it loads that file. Otherwise, it falls back to `App/Config/default.php`.

```php
// app.php -> ['name' => 'anchorv2']
config('app.name');

// default.php -> ['debug' => true]
config('debug');

// default.php -> ['session' => ['timeout' => 3600]]
config('session.timeout');

// database.php -> ['connections' => ['mysql' => ['host' => 'localhost']]]
config('database.connections.mysql.host');
```

## Using Configuration in Classes

### Via Dependency Injection

```php
use Core\Services\ConfigServiceInterface;

class MyService
{
    public function __construct(
        private readonly ConfigServiceInterface $config
    ) {}

    public function doSomething()
    {
        $appName = $this->config->get('app.name');
        $debug = $this->config->get('debug');
    }
}
```

### Via Helper Function

```php
class MyService
{
    public function doSomething()
    {
        $appName = config('app.name');
        $debug = config('debug');
    }
}
```

## Best Practices

- **Never commit .env**: Always add `.env` to `.gitignore`
- **Use .env.example**: Provide a template with dummy values
- **Keep secrets in .env**: API keys, passwords, etc.
- **Use config files for logic**: Complex configuration belongs in config files
- **Production Performance**: Anchor automatically caches configuration. In production mode, file modification checks are skipped for maximum speed.
- **Use descriptive keys**: Make configuration self-documenting
- **Provide defaults**: Always have fallback values

## Configuration Caching

Anchor automatically caches your configuration files to improve performance. This prevents the framework from reloading and parsing multiple PHP files on every request.

### How it Works

- **Production Environment**: Anchor always loads configuration directly from the cache file (`App/storage/cache/config.cache`) for maximum speed.
- **Development Environment**: Anchor checks the modification time of your configuration files in `App/Config/`. If any file has been updated since the cache was created, it automatically refreshes the cache.

The cache is managed internally by the `ConfigService`, so you don't need to manually clear it during development.

## Accessing Configuration Everywhere

### In Controllers

```php
public function index()
{
    $appName = config('app.name');

    // Or via dependency injection
    // public function index(ConfigServiceInterface $config)
}
```

### In Views

```php
<h1><?php echo config('app.name'); ?></h1>
```

### In Services

```php
class EmailService
{
    public function __construct(
        private readonly ConfigServiceInterface $config
    ) {}

    public function send()
    {
        $from = $this->config->get('mail.from.address');
    }
}
```

### In Middleware

```php
public function handle(Request $request, Response $response, Closure $next): mixed
{
    if (config('debug')) {
        // Debug mode logic
    }

    return $next($request, $response);
}
```
