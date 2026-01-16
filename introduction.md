# Introduction

Anchor is a lightweight, module-based PHP framework designed for speed, simplicity, and developer happiness. It provides a robust set of tools for building modern web applications without the overhead of larger frameworks.

## Philosophy

Anchor embraces a maritime metaphor that reflects its core purpose and design philosophy:

### Stability & Foundation

Just as an anchor provides stability to a ship, the **Anchor Framework** gives you a solid, reliable foundation to build upon. It keeps your application grounded with robust architecture, proven patterns, and production ready features, ensuring your codebase remains stable even as your project grows.

### Shipping Code

In software development, we don't just _deploy_ code, we **ship** it. The Anchor Framework is designed to help you confidently ship quality code to production. Every feature, from the ORM to the queue system, is built with production readiness in mind.

### Dock - CLI Tooling

The `dock` command-line interface is your **dock**, the place where you prepare your vessel (application) for its voyage. Just as a ship is maintained, inspected, and provisioned at the dock, the Dock CLI provides essential tools for:

- Generating code scaffolding
- Running migrations and seeders
- Managing queues and scheduled tasks
- Testing and code quality checks (`php dock sail` - your final inspection before deployment)
- Development server management

Together, these elements create a cohesive development experience: **Anchor** provides the foundation, **Dock** equips you with the tools, and you confidently **ship code** to production.

## Requirements

To run Anchor, your server must meet the following requirements:

- **PHP**: >= 8.2
- **Composer**: Dependency Manager
- **Database**: MySQL, PostgreSQL, or SQLite
- **Extensions**:
  - PDO
  - Mbstring
  - OpenSSL
  - Ctype
  - JSON

## Installation

The best way to install Anchor is via Git and Composer.

### Create a New Project

Clone the application skeleton (the template):

```bash
git clone https://github.com/beniyke/anchor my-app
cd my-app
```

### Initial Setup

1. **Initialize**: Run the `dock` command to initialize your application:

   ```bash
   php dock
   ```

2. **Environment**: Copy the example environment file and configure your database:

   ```bash
   cp .env.example .env
   ```

3. **Migrations**: Run the core migrations to prepare the database:
   ```bash
   php dock migration:run
   ```

### Essential System Packages

Now that the foundation is ready, install the core system packages:

```bash
# Install Queue package (Background processing)
php dock package:install Queue --system

# Install Notify package (Alerts and notifications)
php dock package:install Notify --system

# Install Debugger (Essential for development)
php dock package:install Debugger --system
```

### Environment Configuration

Copy the example environment file and configure your database credentials.

```bash
cp .env.example .env
```

Open `.env` and update the settings:

```ini
APP_NAME=AnchorApp
APP_ENV=local
APP_DEBUG=true
APP_URL=localhost:1010

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=anchor_db
DB_USERNAME=root
DB_PASSWORD=
```

### Serve the Application

Start the development server, which handles both the application and the background worker:

```bash
php dock dev
```

Or configure a virtual host in Apache/Nginx pointing to the **project root** directory (where `index.php` is located).

## Key Features

- **Module-Based Architecture**: Organize code by feature, not just file type.
- **Lightweight Core**: Fast request lifecycle with minimal overhead.
- **Powerful ORM**: Eloquent-like syntax for database interactions.
- **Convention over Configuration**: Sensible defaults to get you started quickly.
- **Built-in Tools**: CLI, Migrations, Queues, Mailer, and more.
- **Clear Separation**: `System/` provides the framework foundation (don't modify), `App/` is your workspace (build here).

> Build your application in `App/` using the tools provided by `System/`. If you need framework improvements, contribute via Pull Requests!

## Further Reading

- [Architecture](architecture.md) - High-level framework structure
- [Developer Internals](internals.md) - Deep-core and framework infrastructure
- [Application Lifecycle](lifecycle.md) - How a request flows through Anchor
