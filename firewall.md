# Firewall

The Firewall system provides a robust security layer for your application, handling request throttling, blocking, IP filtering, and audit trails for suspicious activities.

## Firewall Drivers

The system includes several specialized drivers to handle different security aspects.

### AccountFirewall

Protects user accounts against brute-force attacks by throttling attempts based on User ID and device fingerprint.

**Usage:**

```php
use Security\Firewall\Drivers\AccountFirewall;

$firewall = resolve(AccountFirewall::class);

$firewall->user(['id' => $userId])
    ->callback(function($error) {
        // Optional custom error handling
    })
    ->handle();

if ($firewall->isBlocked()) {
    // Handle blocked state
}
```

### AuthFirewall

Protects authentication endpoints (e.g., login) against brute-force attacks by throttling attempts based on an identity field (e.g., email) and IP/device fingerprint.

**Usage:**

```php
use Security\Firewall\Drivers\AuthFirewall;

$firewall = resolve(AuthFirewall::class);

// On failed login
$firewall->fail()->capture();

// On successful login
$firewall->clear()->capture();
```

### ApiRequestFirewall

Secures API endpoints by validating request methods, schemes (HTTP/HTTPS), and content types. It also provides atomic rate limiting for APIs.

**Features:**

- Method validation (GET, POST, etc.)
- Protocol enforcement (HTTPS only)
- Content-Type enforcement (e.g., application/json)
- Rate limiting based on API identifiers

### BlacklistFirewall

Blocks access based on a blacklist of IPs, devices, platforms, or browsers.

**Configuration features:**

- Block specific IPs or dynamic ranges
- Block specific user agents, platforms, or devices
- Wildcard route support

### HostFirewall

Restricts application access to specific allowed hostnames to prevent HTTP Host header attacks.

#### Maintenance Mode

The Firewall handles the application's maintenance mode state. When enabled, it blocks all non-excepted requests with a `503 Service Unavailable` response.

#### Global Maintenance

Enable via environment or config:
```php
'maintenance' => [
    'enable' => true,
    // ...
]
```

#### Granular Maintenance

Anchor introduces **Granular Maintenance**, allowing you to lock down specific system resources without affecting the rest of the application. This is powered by the [Route Context](route-context.md) system.

```php
'maintenance' => [
    'enable' => false, // Global is OFF
    'locked_resources' => [
        'payments', // Only the payments resource is locked
        'billing',
    ],
]
```

When a resource is locked:
- Requests matching the `resource` context (e.g., `App\Billing\Controllers\*`) are blocked.
- Users receive a specific message: "The [resource] module is temporarily under maintenance."
- Non-locked resources continue to function normally.

This is ideal for performing database migrations or updates on specific modules while keeping the main application online.

---

## Troubleshooting

Configure the firewall in `App/Config/firewall.php`.

```php
return [
    // Account Protection
    'account' => [
        'enable' => true,
        'response' => 'Too many attempts. Please try again in {duration}.',
    ],

    // Authentication Protection
    'auth' => [
        'enable' => true,
        'routes' => ['auth/login', 'api/login'],
        'identity' => 'email', // POST field to identify user
        'response' => 'Too many login attempts. Try again in {duration}.',
    ],

    // API Security
    'api-request' => [
        'enable' => true,
        'method' => ['post', 'put', 'delete'], // Methods to check
        'content-type' => [
            'enable' => true,
            'allow' => ['application/json'],
            'response' => ['code' => 415, 'message' => 'Unsupported Media Type'],
        ],
        'scheme' => [
            'enable' => true,
            'allow' => ['https'],
            'response' => ['code' => 403, 'message' => 'HTTPS required'],
        ],
    ],

    // Blacklist
    'blacklist' => [
        'enable' => true,
        'routes' => ['*'], // Apply to all routes
        'block' => [
            'ips' => [
                'specific' => ['192.168.1.50'],
                'dynamic' => ['10.0.'], // Blocks 10.0.*.*
            ],
            'browsers' => ['Internet Explorer'],
            'platforms' => [],
            'devices' => [],
        ],
    ],

    // Host Validation
    'host' => [
        'enable' => true,
        'allow' => ['example.com', 'api.example.com'],
    ],

    // Maintenance Mode
    'maintenance' => [
        'enable' => false,
        'allow' => [
            'routes' => ['api/status'],
            'ips' => ['list' => ['127.0.0.1'], 'ignore' => false],
        ],
    ],

    // Notifications
    'notification' => [
        'mail' => [
            'status' => true,
            'to' => ['admin@example.com'],
        ],
    ],
];
```

## Middleware Usage

To apply firewall rules globally or to specific routes, use the `FirewallMiddleware`.

### Registering Middleware

In `App/Config/middleware.php`:

> [!NOTE]
> As of Anchor 2.6.0, `FirewallMiddleware` is automatically applied to the `web` and `api` groups by the framework core. You only need to register it manually if you are defining a custom middleware group.

The middleware automatically loads the enabled drivers defined in `App/Config/firewall.php` under the `drivers` key.

## Audit Trail & Notifications

The firewall automatically logs suspicious activities (like blocked requests).

- **Caching**: Audit logs are cached for 48 hours (`AUDIT_CACHE_DURATION_SECONDS`) to prevent duplicate alerts for the same event.
- **Notifications**: If enabled in config, an email notification is sent to the admin email address using the `Notifier` system.
- **Deferment**: Email sending is deferred to prevent slowing down the request.

### Audit Log Data

- Timestamp
- Driver Name (e.g., `BlacklistFirewall`)
- Message (e.g., "Blacklisted")
- Source IP
- Browser / Platform / Device
- Request Identifier (if applicable)
