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
- **Extensions**: PDO, Mbstring, OpenSSL, Ctype, JSON, BCMath, ZipArchive, cURL.

## Installation

The recommended way to install Anchor is via the **Anchor Skeleton**. This provides a pre-configured template to get you up and running in seconds.

For detailed instructions on Managed and Standalone installation modes, see the **[Installation Guide](installation.md)**.

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
