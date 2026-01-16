# Directory Structure

Anchor's directory structure is designed to be intuitive, modular, and organized. Here's a comprehensive breakdown.

## Root Directory

```
anchorv2/
├── App/                    # Your application code
├── System/                 # Framework core (don't modify)
├── libs/                   # External, third-party libraries
├── packages/               # Framework-related packages and tools
├── public/                 # Public assets (css, js, images)
├── vendor/                 # Composer dependencies
├── .env                    # Environment variables (not in git)
├── .env.example            # Environment template
├── .gitignore              # Git ignore rules
├── .htaccess               # Apache configuration
├── composer.json           # Composer dependencies
├── dock                    # CLI entry point
├── index.php               # Entry point (Web root)
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
├── Middleware/             # HTTP middleware
├── Models/                 # Shared models
├── Providers/              # Service providers
├── Services/               # Shared services
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
├── cache.php               # Cache configuration
├── command.php             # CLI commands
├── cors.php                # CORS settings
├── database.php            # Database connections
├── default.php             # Default values
├── email_validation.php    # Email validation rules
├── firewall.php            # Security rules
├── functions.php           # Custom functions
├── image.php               # Image processing
├── mail.php                # Email settings
├── menu.php                # Navigation menus
├── middleware.php          # Middleware registry
├── playground.php          # REPL config
├── providers.php           # Service providers
└── route.php               # Routes and middleware
```

**App/Core/**

Base classes for your application:

```
App/Core/
├── BaseController.php      # Base controller
├── Data/                   # Data transfer objects
└── Traits/                 # Reusable traits
```

**App/Middleware/**

HTTP middleware for request/response filtering:

```
App/Middleware/
├── MiddlewareInterface.php
├── Api/                    # API middleware
│   └── ApiAuthMiddleware.php
└── Web/                    # Web middleware
    ├── SessionMiddleware.php
    ├── WebAuthMiddleware.php
    ├── PasswordUpdateMiddleware.php
    └── RedirectIfAuthenticatedMiddleware.php
```

**App/src/**

Your application is organized into modules. Each module is self-contained:

```
App/src/
├── Account/                # Account management module
│   ├── Actions/            # Business logic actions
│   ├── Controllers/        # HTTP controllers
│   ├── Models/             # Database models
│   ├── Notifications/      # Notifications
│   ├── Services/           # Service classes
│   ├── Validations/        # Validation rules
│   └── Views/              # View templates
│       ├── Models/         # View models
│       └── Templates/      # PHP templates
├── Auth/                   # Authentication module
│   ├── Controllers/
│   ├── Notifications/
│   ├── Requests/
│   ├── Services/
│   ├── Tasks/              # Queue tasks
│   ├── Validations/
│   └── Views/
```

**App/storage/**

Application storage for generated files and data:

```
App/storage/
├── build/                  # Build artifacts
├── cache/                  # Application cache
├── database/               # Database files
│   ├── backup/             # Database backups
│   ├── migrations/         # Migration files
│   └── anchor.sqlite       # SQLite database (if used)
├── email-templates/        # Compiled email templates
├── logs/                   # Application logs
```

**Note**: Anchor does NOT store uploaded files or compiled views in `storage/`.

- **Uploaded files**: Typically stored in `public/{dir}/` or handled via cloud storage
- **Compiled views**: Views are plain PHP files, no compilation needed

## System Directory

The `System/` directory contains the **framework core** - the foundation that powers your application.

**CRITICAL: Do NOT Modify System Files**

The `System/` directory is the framework's foundation and must remain untouched because:

- Framework updates will overwrite your changes
- It creates maintenance and debugging nightmares
- It breaks compatibility with framework improvements

**Need to improve the framework?**

- Submit a Pull Request to the Anchor Framework repository
- Open a GitHub Issue to discuss the feature
- Contribute to making Anchor better for everyone

**Focus on `App/` for your project** - that's your workspace!

```
System/
├── Cli/                    # CLI commands
│   └── Commands/
│       └── Database/       # Database commands
├── Core/                   # Core framework
│   ├── App/                # Application bootstrap
│   ├── Globals/            # Global helper functions
│   ├── Ioc/                # IoC container
│   ├── Route/              # Routing system
│   ├── Services/           # Core services
│   └── Views/              # View engine
├── Database/               # Database layer
│   ├── Migration/          # Migration system
│   ├── Query/              # Query builder
│   ├── Relations/          # ORM relationships
│   ├── Schema/             # Schema builder
│   └── Traits/             # Database traits
├── Debugger/               # Debug bar
├── Helpers/                # Helper classes
│   ├── Data/               # Data helpers
│   ├── Encryption/         # Encryption
│   ├── File/               # File system
│   ├── Http/               # HTTP helpers
│   └── String/             # String helpers
├── Mail/                   # Email system
├── Notify/                 # Notifications
├── Queue/                  # Queue system
└── Security/               # Security features
    └── Firewall/           # Firewall system
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
└── uploads/                # User uploads (if stored locally)
```

**Important**: Point your web server to the project root directory, not `public/`.

## Module Structure Example

A typical module (e.g., `App/src/Tweets/`):

```
Tweets/
├── Actions/                # Business logic
│   └── CreateTweetAction.php
├── Controllers/            # HTTP controllers
│   └── TweetController.php
├── Models/                 # Database models
│   └── Tweet.php
├── Services/               # Service layer
│   └── TweetService.php
├── Validations/            # Validation rules
│   └── Form/
│       └── TweetFormRequestValidation.php
└── Views/                  # View layer
    ├── Models/             # View models
    │   └── TweetViewModel.php
    └── Templates/          # PHP templates
        ├── index.php
        ├── show.php
        └── create.php
```

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
| `vendor/`      | Composer packages            | Auto-generated      |

## Best Practices

- **Keep modules self-contained**: Each module should have its own controllers, models, views
- **Never modify `System/`**:
  - The framework core is your foundation, not your workspace
  - Framework updates will overwrite any changes you make
  - If you need framework improvements, submit a PR to the repository
  - **Focus on building in `App/`** - that's where your application lives
- **Use `storage/` for generated files**: Logs, cache, database files

- **Store uploads in `public/`**: Or use cloud storage (S3, etc.)
- **Follow naming conventions**: Controllers end in `Controller`, Models are singular
- **Organize by feature**: Group related files in modules, not by type globally

## Contributing to the Framework

If you discover a bug or want to add a feature to Anchor itself:

- **Don't modify `System/` directly** - it breaks updates
- **Open a GitHub Issue** - discuss the improvement
- **Submit a Pull Request** - contribute to the framework
- **Help the community** - make Anchor better for everyone

> `System/` is the framework foundation, `App/` is where you build your ship!
