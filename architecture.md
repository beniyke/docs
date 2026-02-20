# Framework Architecture

Anchor is built on a **Modular MVC (Model-View-Controller)** architecture, enhanced with a robust **Service Container** for dependency injection. It emphasizes separation of concerns, modularity, and explicit service registration.

## Overview

The framework handles requests through a linear pipeline:
`Entry Point` -> `Kernel` -> `Router` -> `Middleware` -> `Controller` -> `Response`

- **Core Kernel**: The engine driving the application.
- **Service Container**: The "brain" managing class dependencies.
- **Service Providers**: The "bootstrappers" configuring the application.
- **Modules**: Self-contained feature sets (HMVC style).

Anchor has a clear separation between the **framework foundation** (`System/`) and **your application** (`App/`):

### System Directory

### Framework Foundation

The `System/` directory contains the **core framework code** that powers Anchor. This is the foundation that provides:

- Routing, middleware, and request handling
- Database query builder and ORM
- Queue system, mail, notifications
- CLI commands and code generators
- Security features and helpers

The `System/` directory is the framework's foundation and should remain untouched. Modifying core files will:

- Break framework updates
- Create maintenance nightmares
- Cause conflicts when pulling framework improvements
- Make your application difficult to debug

**If you need to improve or add features to the framework itself:**

- Submit a Pull Request to the Anchor Framework repository
- Discuss the feature in GitHub Issues
- Contribute to the framework's evolution

### App Directory

### Your Application

The `App/` directory is **your workspace** where you build your application:

- Create modules for your business logic
- Define routes, controllers, and services
- Build views and view models
- Configure your application behavior
- Write middleware specific to your needs

**This is where you focus your development efforts.** The framework provides the tools in `System/`, and you use them to build your application in `App/`.

### The Anchor Philosophy

| Feature          | **System Directory** (Framework) | **App Directory** (Application) |
| :--------------- | :------------------------------- | :------------------------------ |
| **Role**         | Foundation & Core Stability      | Implementation & Business Logic |
| **Status**       | **Locked** (Vendor Control)      | **Open** (User Control)         |
| **Modification** | **Do Not Modify**                | **Full Control**                |
| **Updates**      | Handled by Framework Updates     | Managed by You                  |

## The Core Kernel

The **Kernel** (`System\Core\Kernel`) is the central orchestrator. It is responsible for:

- Bootstrapping the application.
- Loading Service Providers.
- Handling exceptions globally.
- Terminating the application (sending response, running deferred tasks).

The application is then handed off to one of two handlers:

- **Http Handler** (`Core\App`): Handles web requests via `index.php`.
- **Console Handler** (`Core\Console`): Handles CLI commands via the `dock` utility.

## Dependency Injection

Anchor uses a **Service Container** (`Core\Ioc\Container`) to manage class dependencies. Instead of creating objects manually (`new Class`), you ask the container to provide them.

- **Automatic Injection**: The container analyzes class constructors and automatically injects dependencies.
- **Singletons**: Shared instances reused throughout the request (e.g., `Database`, `Config`).
- **Interface Binding**: You can swap implementations by binding an Interface to a concrete Class.

## Service Providers

**Service Providers** are the primary mechanism for configuring your application. They are listed in `App/Config/providers.php`.

- **Register Phase** (`register()`): Only bind things into the container. Fast and lightweight.
- **Boot Phase** (`boot()`): Perform work after all services are registered (e.g., event listeners, custom configuration).

## Middleware Pipeline

HTTP Requests pass through a stack of **Middleware** before reaching your controller. This implements the "Chain of Responsibility" pattern.

```text
Request -> [FirewallMiddleware] -> [WebAuthMiddleware] -> [YourController]
                                                                |
Response <- [FirewallMiddleware] <- [WebAuthMiddleware] <- [YourController]
```

Middleware filter requests (security, auth) and modify responses (headers, logging).

## Modular Structure

### App/src structure

Anchor encourages a **Domain-Driven** or **Modular** structure in `App/src`. Instead of grouping by file type (all controllers in one folder), you group by feature:

```text
App/src/
└── Account/           <-- "Account" Module
    ├── Controllers/   <-- Controllers specific to Account
    ├── Services/      <-- Business logic for Account
    └── Models/        <-- Data models for Account
```

This ensures that as the application grows, features stay encapsulated and maintainable. Package-specific logic for core models is injected at runtime via **Macros** to prevent cross-modular pollution.

## Database Layer

- **Query Builder**: A fluent interface for constructing SQL queries securely.
- **ORM (Active Record)**: Models represent database tables (`User::find(1)`).
- **Migrations**: Version control for database schema.

## Summary

| Component     | Responsibility                         |
| :------------ | :------------------------------------- |
| **Kernel**    | Orchestrates the request lifecycle.    |
| **Container** | Manages dependencies and injection.    |
| **Providers** | Bootstraps configuration and bindings. |
| **Modules**   | Encapsulates business logic by domain. |

For detailed implementation guides, see [lifecycle](lifecycle) and [container](container).
