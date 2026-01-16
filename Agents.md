# AI Agent Guide

Welcome, Agent. This document defines the rules, patterns, and best practices for developing within the **Anchor Framework**. Adherence to these guidelines is mandatory to maintain the framework's integrity, testability, and "mutation-resistant" nature.

## The Anchor Philosophy

- **Stability**: Anchor provides a solid foundation. Don't fight the framework; use its built-in abstractions.
- **Shipping**: Code is not just deployed; it's **shipped**. It must be production-ready, performant, secure, and "seaworthy."
- **Analytic Robustness**: Every feature should include robust analytics/logging to ensure visibility into production behavior.
- **Separation of Concerns**:
  - `System/`: Framework core. **DO NOT MODIFY**.
  - `App/`: Your workspace. Implement business logic here.
  - `packages/`: Modular extensions. Each package is a self-contained ecosystem.

## Architectural Principles

### Modular MVC (HMVC)

- Organize by **Feature/Domain**, not just file type.
- Each module/package should be encapsulated with its own Controllers, Services, Models etc.
- **Package Isolation**: No package in `packages/` may contain test files or a `Tests/` directory. (See [package-management.md](package-management.md) for full package specifications).
- **Cross-Package Interaction**: Packages must **never** import or manipulate models belonging to other packages. All interactions must pass through **Service Facades** or **Event Hooks**.
- **Integration Guard**: Before integrating with other packages, always check if they exist/are enabled and provide a graceful fallback.
- **Clear SOC**: Maintain a strict Separation of Concerns. Every Command should interact with a Service; logic should not reside in the Command itself.

### Naming & Directory Conventions

Components must follow strict naming and directory patterns aligned with the framework code:

- **Events**: `ExampleEvent` (must reside in an `Events/` directory).
- **Listeners**: `ExampleListener` (must reside in a `Listeners/` directory).
- **Services**: `ExampleService` (must reside in a `Services/` directory).
- **Notifications**: `ExampleNotification` (must reside in a `Notifications/` directory).
- **Controllers**: `ExampleController` (must reside in a `Controllers/` directory).
- **ViewModels**: `ExampleViewModel` (must reside in a `Views/Models/` or `ViewModels/` directory).
- **Commands**: `ExampleCommand` (must reside in a `Commands/` directory).
- **Requests**: `ExampleRequest` (must reside in a `Requests/` directory).
- **Channels**: `ExampleChannel` (must reside in a `Channels/` directory).
- **Models**: `Example` (must reside in a `Models/` directory). Note: Models do **not** use a suffix.
- **Facades**: `Example` (must reside in a `Facades/` directory).
- **Traits**: `HasExample` or `InteractsWithExample` (must reside in a `Traits/` directory).
- **Interfaces/Contracts**: `ExampleInterface` (reside in `Contracts/` or `Interfaces/`).
- **Actions**: `ExampleAction` (must reside in an `Actions/` directory).
- **Enforcement**: If a class exists in these directories but lacks the suffix, it **must be renamed**, and all references updated.
- **Pattern Matching**: Refer to existing framework code (e.g., in `System/` or stable `packages/`) for other patterns.

### Dependency Injection (DI)

- **Constructor Injection**: Mandatory for Services, Packages, and core logic. Improves testability and makes dependencies explicit.
- **Global Helpers**: `config()`, `session()`, `request()`, etc. are acceptable in **Controllers** and **Views** for better Developer Experience (DX).
- **Golden Rule**: If you need to mock it in a unit test, inject it.

### Service Container & Providers

- Bind interfaces to concrete implementations in Service Providers (`register()`).
- Use `singleton()` for shared instances (DB, Config) and `bind()` for fresh instances (Validators, DTOs).

## Coding Standards & Quality

### Strict Typing

- Every PHP file **MUST** declare strict types: `declare(strict_types=1);`.
- Use native type hints for parameters, return types, and class properties.
- Use PHPDoc `/** @var ... */` only for complex types (e.g., generics `array<string, User>`).

### Static Analysis (PHPStan)

- The codebase targets **PHPStan Level 5**.
- Avoid `mixed` types where possible.
- Use guard clauses or null-coalescing to handle potentially nullable values.

### I/O Abstractions

- **Avoid native I/O functions** (`file_exists`, `mkdir`, `copy`) in core logic/packages.
- Use framework interfaces: `PathResolverInterface`, `FileMetaInterface`, `FileManipulationInterface`.
- **Reason**: Native functions break unit test isolation and cross-platform compatibility.

### Code Style (PHP)

- **Document Intent**: Document the _why_ for non-obvious code, not just the _what_.
- **Purpose Docblocks**: Every class, trait, interface, or enum under `App/` or `packages/` must have a top-level PHPDoc explaining:
  - Why the file exists.
  - Why the logic was extracted there (vs. inlining).
  - The "contract" or what callers should rely on (if non-obvious).
- **Import Namespaces**: Strictly prioritize `use` imports over fully qualified class names (FQCNs) in all package code. Do not rely on implicit or global imports.
- **Explicit Naming**: Avoid ambiguous names. No one-letter variables unless they are extremely local (e.g., in a short closure) and obvious.
- **Guard Clauses**: Use guard clauses to reduce nesting depth.
- **Fluent Methods**: Provide fluent interfaces (returning `$this`) in Builders and Services where it improves Developer Experience (DX).
- **No Debugging**: Never commit `dd()`, `dump()`, or other debugging helpers.
- **No `final`**: Do not use the `final` keyword; prioritize extensibility.
- **No Suppression**: Never use the `@` error suppression operator. Use explicit error handling and document why if it's unavoidable.
- **Visibility**: Default to `protected` for non-public methods and properties unless there is a reason for `private`.

### Security Baseline

- **No Globals**: Never use native PHP globals (`$_GET`, `$_POST`, `$_SESSION`, `$_FILES`). Always use the framework's `Request` object or relevant abstractions.
- **Validation First**: Every piece of user-provided data **must** be validated before it is used to interact with the database or external services.
- **Mass Assignment**: Be explicit about `$fillable` fields in Models; never use `$guarded = []`.

### Performance & Scalability

- **Eager Loading**: Always use `.with()` or `.load()` to prevent the N+1 query problem when accessing relationships in loops.
- **Model Factories**: Every model **must** have a corresponding Factory in `tests/System/Fixtures/Factories/` or the package's `Fixtures/` directory to ensure consistent test data.
- **Queued Tasks**: Offload heavy or time-consuming operations (e.g., sending emails, processing large files) to background Jobs.

### Error Handling

- **Type Safety**: Standardize on `Throwable` for catch blocks unless you are targeting a specific custom exception.
- **Custom Exceptions**: Create high-level domain exceptions (residing in `Exceptions/`) to represent meaningful business failures that need special handling.

### Database & Migrations

- **Table Naming**: Table names must be in **singular** form (e.g., `user`, `order`).
- **One Migration, One Table**: No multi-table migrations. Every table should belong to its own migration file.
- **Timestamps**: Use `dateTimeStamps()` over standard `timestamps()` for better precision/consistency across drivers.
- **Reversibility**: Migrations should be reversible (`down()` method implemented) whenever possible.
- **Immutability**: Never edit a migration file after it has been merged/deployed. Always create a new migration to modify the existing schema.

## Testing Guidelines (Pest 3.x)

Testing is not optional; it's the core of the Anchor Framework. All tests must comply with the standards outlined in [testing.md](testing.md).

- **Testing Framework**: Use **Pest exclusively** for all tests. PHPUnit must not be used.
- **Centralized Testing**: All tests for packages must reside under the global `tests/` directory, following the structure and conventions defined in [testing.md](testing.md).

### Unified Directory Mirroring

- Your `tests/` directory **MUST** mirror your `src/` directory.
- `Unit/`: Isolated logic. **NO** database, **NO** container.
- `Feature/`: Request-to-response journeys. Transactional database allowed.
- `Architecture/`: Structural rules using `arch()`.

### Mutation Resistance

- **Goal**: 100% Mutation Score for Unit tests.
- Use `with()` datasets to cover **boundaries** (e.g., 17, 18, 19 for age 18 check) and **edge cases** (null, empty, max int).
- Kill mutants like `>` to `>=` by testing the exact boundary value.

### Database Rules

- **No Inline Migrations**: Never define schema in test files.
- Use the `RefreshDatabase` trait.
- Use `Fixtures/` for test-only models and `Factories/` for data seeding.

### Architecture Guardrails

- Every module/package **MUST** have an `ArchTest.php`.
- **Naming Enforcement**: Include ArchTests to automatically verify that classes in `Events/`, `Listeners/`, `Services/`, and `Notifications/` have the required suffixes.
- Enforce naming conventions (e.g., `*ServiceProvider`, `*Command`, `*Test`).
- Prevent layer breaches (e.g., "System should not depend on App", "Packages should not import other package models").

## Tooling & Workflow

Use the `dock` CLI for all lifecycle tasks:

- `php dock format`: Auto-fix coding style and cleanup comments.
- `php dock inspect`: Run Pint and PHPStan to catch issues.
- `php dock sail`: **The Ultimate Gatekeeper**. Runs style, static analysis, and tests in parallel. Run this before declaring a task complete.
- `php dock package:install {Name}`: Use `--system --force` for core framework packages.

## Summary for Agents

- **Strict Types** on every file.
- **Abstractions** for File I/O.
- **Constructor Injection** for logic.
- **Mirror Tests** for every class.
- **100% MSI** is the target.
- **`php dock sail`** is your best friend.
