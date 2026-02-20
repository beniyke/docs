# Developer Internals

This document covers the deep-core architectural decisions and infrastructure that power the Anchor Framework. It is intended for framework contributors and developers who want to understand the "magic" behind the scenes.

## Decoupled Core (Adapters Pattern)

Anchor adheres to a strict "Protected Core" philosophy. To ensure the framework remains testable and environment-agnostic, we avoid direct calls to native PHP functions or global state whenever possible.

Instead, the framework uses **Core Adapters** located in `System/Core/Support/Adapters`.

### Filesystem Abstraction

Every core CLI command or system service that interacts with the disk uses the `FileMetaInterface` and `FileReadWriteInterface`.

- **Why?** This allows us to swap the local filesystem for a virtual filesystem during testing, or even a cloud-based filesystem without changing a single line of procedural code in our commands.

### OS & SAPI Detection

Instead of using `PHP_OS` or `PHP_SAPI` directly, services inject `OSCheckerInterface` and `SapiInterface`.

- **Example**: The `php dock dev` command uses the `OSChecker` to determine whether to use Windows-specific or Unix-specific process signals for restarting the server.

## Monorepo Infrastructure

Anchor is developed in a **Monorepo** and automatically split into individual read-only repositories for distribution.

### Metadata Synchronization

To maintain a single source of truth, we use automated "Runners" that execute before the monorepo is split:

- **Version Sync (`version:sync`)**:

  - The framework version is defined in a single `version.txt` in the root.
  - The sync command reads this file and "bakes" it into the `App::VERSION` constant in `System/Core/App.php`.
  - This ensures that even when a package is installed in "Managed" mode via Composer, it knows exactly which framework version it is running on without hitting the disk again.

- **Docs Sync (`docs:sync`)**:
  - Primary documentation lives in the central `docs/` directory.
  - Before splitting, the framework finds the corresponding `.md` file for each package (e.g., `docs/pay.md` -> `packages/Pay/README.md`) and syncs the content.
  - This ensures that individual repositories have up-to-date READMEs despite the documentation being managed centrally.

### The Splitting Strategy

GitHub Actions partition the monorepo into:

- **`beniyke/framework`**: The core `System/` directory.
- **`beniyke/tests`**: The central testing suite.
- **`beniyke/docs`**: The central documentation repository.
- **`beniyke/anchor-skeleton`**: The official application starter project.
- **`beniyke/{package-name}`**: Individual repositories for every package in `packages/`.

## The "Smart Skeleton" Architecture

Anchor provides a unique installation flexibility: **Managed** vs **Portable**.

### Managed Mode (Composer)

The standard modern PHP approach. `beniyke/anchor-skeleton` requires `beniyke/framework` via Composer. The `vendor/` directory manages all dependencies.

### Portable Mode (Standalone)

Designed for low-resource environments or rapid prototyping where Composer access might be restricted.

- The `dock` executable acts as a **Smart Bootstrap**. It automatically detects if the framework is missing (e.g., a fresh skeleton) and offers an interactive provisioning flow directly in the terminal.
- It can "Self-Hydrate" by downloading the `System` and `libs` folders directly from GitHub releases as a zip.
- The `System` directory is kept "Vendor-Free" to facilitate this mode, using the built-in `libs` directory for small, essential external bridges.

## Advanced Formatter Logic

The `CodeFormatter` (`System/Formatter/CodeFormatter.php`) is an opinionated cleanup engine. It doesn't just fix spacing; it simplifies the "intent" of the code.

### Redundant Docblock Removal

The formatter will remove a docblock if it adds zero value to a modern PHP signature.

**Before:**

```php
/**
 * Set the name
 *
 * @param string $name
 * @return void
 */
public function setName(string $name): void
```

**After (Cleanup):**

```php
public function setName(string $name): void
```

### Explanatory vs. Narrative Comments

The formatter uses a **Heuristic Narrative Engine** to decide which comments to keep:

- **Keep**: "This is a workaround for a documented bug in PHP 8.2..." (Found keywords: _because_, _since_, _workaround_).
- **Remove**: "// Set name" right above `setName()`.

## REPL Lifecycle & Auto-Imports

The Application Playground (`php dock playground`) uses a persistent shell configuration.

- **Context Injection**: The `PlaygroundCommand` automatically binds the running `Container` and `Config` instances into the shell.
- **Auto-Imports**: Defined in `App/Config/playground.php`. This allows you to reference your Models or Services using their Short Name (e.g., `User::all()`) instead of the Full Namespace, making it feel like a truly interactive exploration tool.

## State Management & Reset

In long-running environments (like Queue Workers or the built-in development server), PHP's shared-nothing architecture is challenged by persistent memory growth. Anchor solves this via a robust **Termination Workflow**.

### TerminableInterface

Services that accumulate data over time (e.g., query logs, message collectors) should implement the `Core\Contracts\TerminableInterface`.

```php
interface TerminableInterface
{
    public function terminate(): void;
}
```

### The Termination Flow

When `Kernel::terminate()` is called:

- It retrieves all loaded service provider instances from the `ProviderManager`.
- Any provider implementing `TerminableInterface` has its `terminate()` method called.
  - **`Debugger`**: Resets the `DebugBar` collector data.
  - **`Deferrer`**: Clears any pending deferred task payloads.
- Static framework state is explicitly cleared:
  - **`Database\Connection`**: Clears the query log.
  - **`Helpers\Benchmark`**: Resets all timers and memory markers.
  - **`Core\Route\UrlResolver`**: Clears static route resolution and controller caches.
  - **`Core\Event`**: Clears event fakes (persists listeners for worker stability).
  - **`Helpers\String\Inflector`**: Clears the pluralization/singularization cache.

This ensures that even in long-running processes (like Queue Workers), the framework remains stable. The `Event::reset()` call specifically clears any event fakes used in testing while preserving the registered listeners, ensuring the worker remains functional for subsequent jobs without memory growth from mocked state.
