# Directory Structure

Anchor's directory structure is designed to be intuitive, modular, and organized. Here's a comprehensive breakdown.

## Root Directory

```
anchorv2/
├── App/                    # Your application code
├── System/                 # Framework core (don't modify)
├── docs/                   # Framework documentation
├── libs/                   # External, third-party libraries
├── packages/               # Framework-related packages and tools
├── public/                 # Public assets (css, js, images)
├── tests/                  # Application and framework tests
├── vendor/                 # Composer dependencies
├── .env                    # Environment variables (not in git)
├── .env.example            # Environment template
├── .gitignore              # Git ignore rules
├── .htaccess               # Apache configuration
├── README.md               # Project documentation
├── composer.json           # Composer dependencies
├── cron.php                # Task scheduling entry point
├── dock                    # CLI entry point
├── index.php               # Entry point (Web root)
├── server.php              # PHP built-in server script
└── worker                  # Queue worker script
```

## App Directory

The `App` directory contains all your application-specific code.

```
App/
├── Channels/               # Broadcasting channels
├── Commands/               # Console commands
├── Config/                 # Configuration files
├── Core/                   # Base application classes
├── Enums/                  # Enumerations
├── Events/                 # Event classes
├── Helpers/                # Custom helper classes
├── Listeners/              # Event listeners
├── Middleware/             # HTTP middleware
├── Models/                 # Shared models
├── Providers/              # Service providers
├── Services/               # Shared services
├── Tasks/                  # Shared queue tasks
├── Views/                  # Shared views
├── src/                    # Application modules
└── storage/                # Application storage
```

**App/Config/**

Configuration files that define application behavior:

```
App/Config/
├── api.php                 # API settings
├── app.php                 # General app settings
├── auth.php                # Authentication guards and sources
├── cache.php               # Cache configuration
├── command.php             # CLI commands
├── cors.php                # CORS settings
├── database.php            # Database connections
├── default.php             # Default values
├── email_validation.php    # Email validation rules
├── filesystems.php         # Filesystem disks and drivers
├── firewall.php            # Security rules
├── functions.php           # Custom functions
├── image.php               # Image processing
├── mail.php                # Email settings
├── middleware.php          # Middleware registry
├── permissions.php         # System privilege registry
├── playground.php          # REPL config
├── providers.php           # Service providers
└── route.php               # Routes and middleware
```

**App/Core/**

Base classes for your application:

```
App/Core/
├── BaseAction.php          # Base business logic action
├── BaseController.php      # Base controller
├── BaseRequest.php         # Base request handler
├── BaseRequestValidation.php # Base validation logic
├── BaseResource.php        # Base API resource
├── Data/                   # Data transfer objects
└── Traits/                 # Reusable traits
```

**App/Middleware/**

HTTP middleware for request/response filtering:

```
App/Middleware/
├── Api/                    # API middleware
│   ├── ApiAuthMiddleware.php
│   └── CorsMiddleware.php
└── Web/                    # Web middleware
    ├── WebAuthMiddleware.php
    ├── PasswordUpdateMiddleware.php
    ├── RedirectIfAuthenticatedMiddleware.php
    └── SecurityHeadersMiddleware.php
```

*Note: Core middleware like `SessionMiddleware` is located in `System/Core/Middleware/`.*

**App/src/**

Your application is organized into modules. Each module is self-contained:

```
App/src/
├── Account/                # Account management module
├── Auth/                   # Authentication module
├── Docs/                   # Documentation module
└── Website/                # Website/Frontend module
```

**App/storage/**

Application storage for generated files and data:

```
App/storage/
├── app/                    # General application storage
├── build/                  # Build artifacts
├── cache/                  # Application cache
├── database/               # Database files
│   ├── migrations/         # Migration files
│   └── anchor.sqlite       # SQLite database (if used)
├── email-templates/        # Compiled email templates
├── logs/                   # Application logs
├── query/                  # Database query logs
└── testing/                # Test-specific storage
```

**Note**: Anchor does NOT store uploaded files or compiled views in `storage/`.

- **Uploaded files**: Typically stored in `public/{dir}/` or handled via cloud storage
- **Compiled views**: Views are plain PHP files, no compilation needed

## System Directory

The `System/` directory contains the **framework core** - the foundation that powers your application.

**CRITICAL: Do NOT Modify System Files**

```
System/
├── Cli/                    # CLI commands
├── Core/                   # Core framework logic
│   ├── App/                # Application bootstrap
│   ├── Middleware/         # Core middleware (Session, etc.)
│   ├── Route/              # Routing system
│   └── Views/              # View engine
├── Cron/                   # Task scheduling
├── Database/               # Database layer
├── Debugger/               # Debug bar
├── Defer/                  # Deferred execution
├── Exceptions/             # Exception handling
├── Formatter/              # Data formatting
├── Helpers/                # Core helper classes
├── Mail/                   # Email system
├── Notify/                 # Notifications
├── Package/                # Package management
├── Queue/                  # Queue system
├── Security/               # Security and Authentication
│   ├── Auth/               # Core Authentication (Service, Result)
│   └── Firewall/           # Firewall and security rules
└── Testing/                # Test framework core
```

## Public Directory

The web server document root. All HTTP requests go through `index.php`.

```
public/
├── assets/                 # Static assets
│   ├── css/                # Stylesheets
│   ├── js/                 # JavaScript
│   ├── images/             # Images
│   └── fonts/              # Fonts
├── uploads/                # User uploads (if stored locally)
└── sent-mail/              # Local mail trap (dev environment only)
```

**Important**: Point your web server to the project root directory, not `public/`.

## Key Directories Summary

| Directory      | Purpose                      | Modify?             |
| -------------- | ---------------------------- | ------------------- |
| `App/`         | Your application code        | ✓ Yes               |
| `App/Config/`  | Configuration files          | ✓ Yes               |
| `App/src/`     | Application modules          | ✓ Yes               |
| `App/storage/` | Generated files, logs, cache | Auto-generated      |
| `System/`      | Framework core               | ✗ No                |
| `libs/`        | External libraries           | ✓ Yes (additions)   |
| `packages/`    | Framework packages           | ✓ Yes (installable) |
| `public/`      | Web root, static assets      | ✓ Yes (assets only) |
| `tests/`       | Application tests            | ✓ Yes               |
| `vendor/`      | Composer packages            | Auto-generated      |

## Best Practices

- **Keep modules self-contained**: Each module should have its own controllers, models, and views.
- **Never modify `System/`**: Use `App/` for your project logic. Framework updates will overwrite `System/` changes.
- **Use `storage/` for generated files**: Logs, cache, and database files.
- **Follow naming conventions**: Controllers end in `Controller`, Models are singular.
- **Organize by feature**: Group related files in modules, not by type globally.
