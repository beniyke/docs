# Tenancy

Anchor provides a robust multi-tenancy system out of the box, allowing you to serve multiple customers (tenants) from a single installation. Each tenant gets their own isolated environment, ensuring data privacy and security.

## Installation

Tenancy is a **core package** that requires installation before use.

### Install the Package

```bash
php dock package:install Tenancy --packages
```

This will automatically:

1. Copy configuration files to `App/Config/`
2. Register the `TenancyServiceProvider`
3. Register the `TenantIdentificationMiddleware`
4. Run core migrations (creates `tenant` table in the central database)

> Ensure your central database is configured in `App/Config/database.php` before running the installation.

## Architecture

The tenancy package defaults to a **Separate Database Strategy**, where each tenant has their own dedicated database.

- **Tenant Identification**: Tenants are identified by subdomain (e.g., `acme.yourdomain.com`).
- **Isolation**: Each tenant has a unique database, user, and password.
- **Auto-Provisioning**: Databases are automatically created and migrated during tenant creation.
- **Context Switching**: The system automatically switches database connections based on the identified tenant subdomain.

## Configuration

The tenancy configuration is located in `App/Config/tenant.php`.

```php
return [
    'enabled' => true,

    'database' => [
        'driver' => 'mysql',
        'prefix_pattern' => 'tenant_', // e.g., tenant_acme
        'default_host' => '127.0.0.1',
        'default_port' => '3306',
        'migrations_path' => 'App/storage/database/migrations',
    ],

    'cache' => [
        'ttl' => 3600,
        'key_prefix' => 'tenant:',
    ],

    'excluded_subdomains' => [
        'admin', 'api', 'www', 'mail', 'localhost'
    ],

    'excluded_paths' => [
        '/health',
        '/status',
        '/_debug',
    ],
];
```

## Usage

### Provisioning a Tenant

The `TenantProvisioningService` handles the full lifecycle of tenant creation.

```php
use Tenancy\Services\TenantProvisioningService;

$service = new TenantProvisioningService();

$tenant = $service->create([
    'name' => 'Acme Corp',
    'subdomain' => 'acme',
    'email' => 'admin@acme.com',
    'plan' => 'pro',
    'seed' => true, // Run seeders after migration
]);
```

**What happens during provisioning:**

1. **Validation**: Checks subdomain format and uniqueness.
2. **Persistence**: Creates a record in the central `tenant` table.
3. **Database Creation**: Creates the physical database and a dedicated user.
4. **Migration**: Runs migrations from the configured `migrations_path`.
5. **Seeding**: Executes the default `TenantSeeder` (if `seed` is true).

### Automatic Context Identification

Tenancy identification is handled via middleware. By default, the installer registers `TenantIdentificationMiddleware` in `App/Config/middleware.php`:

```php
// App/Config/middleware.php
return [
    'web' => [
        // ...
        Tenancy\Middleware\TenantIdentificationMiddleware::class,
    ],
    'api' => [
        // ...
        Tenancy\Middleware\TenantIdentificationMiddleware::class,
    ],
];
```

#### Controlling Application

In Anchor, middleware is applied to groups (like `web` or `api`). To exclude specific paths from tenancy identification (e.g., public pages that don't need a tenant context), use the `excluded_paths` option in `App/Config/tenant.php`.

### Manual Context Switching

In some cases (CLI jobs, background tasks), you may need to switch context manually using the `Tenancy` facade:

```php
use Tenancy\Tenancy;
use Tenancy\Models\Tenant;

// Switch to a specific tenant
$tenant = Tenant::where('subdomain', 'acme')->first();
Tenancy::setContext($tenant);

// Your code now runs in acme's database...

// Reset to central context
Tenancy::reset();
```

## Console Commands

`tenant:create` Create a new tenant from the CLI:

```bash
php dock tenant:create "Acme Corp" acme admin@acme.com pro
```

`tenant:migrate` Run migrations for one or all tenants:

```bash
# Migrate all tenants
php dock tenant:migrate

# Migrate a specific tenant
php dock tenant:migrate --tenant=acme

# Fresh migration with seeding
php dock tenant:migrate --fresh --seed
```

`tenant:list` List all registered tenants and their status:

```bash
php dock tenant:list
```

## Security

### Database Password Encryption

Tenant database passwords are encrypted at rest using `APP_KEY`. The `Tenant` model handles this automatically via attributes:

```php
$tenant->db_password = 'secret-password'; // Encrypted before saving
echo $tenant->db_password; // Decrypted when accessed
```

### Subdomain Protection

Subdomains are strictly validated against injection patterns and a list of `excluded_subdomains`. Only lowercase alphanumeric characters and hyphens are allowed.
