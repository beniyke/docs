# Versioning Policy

Anchor Framework follows [Semantic Versioning 2.0.0](https://semver.org/) (SemVer). This ensures that developers can predict the impact of updating the framework or its packages.

## Version Format

Versions are defined as `MAJOR.MINOR.PATCH`:

- **MAJOR**: Breaking changes that require manual intervention or code updates.
- **MINOR**: New features and enhancements that are backward-compatible.
- **PATCH**: Backward-compatible bug fixes and security updates.

## Determining the Version

### Major Version (X.0.0)

Increment the **Major** version when you make incompatible API changes.

**Use Cases:**

- **Renaming/Removing Public Methods**: Deleting a method from a core Service or Facade.
- **Changing Signature**: Adding a required parameter to an existing public method.
- **Dependency Upgrades**: Increasing the minimum PHP version requirement (e.g., PHP 8.2 to 8.4).
- **Architectural Shift**: Changing how modules are loaded or how the Container resolves dependencies.

> [!WARNING]
> Major updates should always include an `UPGRADE.md` or clear migration instructions.

### Minor Version (0.X.0)

Increment the **Minor** version when you add functionality in a backward-compatible manner.

**Use Cases:**

- **New Features**: Adding a new `Mail` driver or a new `Validation` rule.
- **New Helpers**: Introducing new global helper functions like `str_slug()`.
- **Deprecations**: Marking an old method as `@deprecated` while keeping it functional.
- **Enhancements**: Adding optional parameters to existing methods.

### Patch Version (0.0.X)

Increment the **Patch** version when you make backward-compatible bug fixes.

**Use Cases:**

- **Bug Fixes**: Fixing a syntax error, a logical flaw, or a security vulnerability.
- **Documentation**: Fixing typos or improving clarity in the docs.
- **Performance**: Optimizing an internal loop without changing the external behavior.
- **CI/CD**: Updating GitHub Actions or build scripts.

## Internal Implementation

### Version Source of Truth

The version is primarily tracked in two places:

- **`version.txt`**: Located at the project root for easy reading by CI/CD and deployment scripts.
- **`System/Core/App.php`**: Contains the `public const VERSION` used at runtime.

### Syncing Versions

To ensure consistency across the framework, use the built-in CLI command:

```bash
php dock version:sync
```

This command reads the value from `version.txt` and updates the constant in `App.php`.

### Release Workflow

- Update `version.txt`.
- Run `php dock version:sync`.
- Commit both files.
- Tag the commit in Git (e.g., `git tag v2.2.1`).
