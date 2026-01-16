# ViewModels

ViewModels are a design pattern used in Anchor to encapsulate presentation logic and data preparation for Views. Instead of passing raw arrays or complex models directly to a view, you pass a dedicated `ViewModel` object.

## Why use ViewModels?

- **Separation of Concerns**: Moves presentation logic (formatting dates, checking errors, conditional classes) out of the View and Controller.
- **Type Safety**: ViewModels are classes with typed properties and methods, providing better IDE support than associative arrays.
- **Testability**: You can easily unit test a ViewModel without needing to render a view.
- **Reusability**: Shared presentation logic can be reused across different views.

## Implementation

A ViewModel is typically a simple PHP class (often `readonly`) that takes dependencies in its constructor and exposes public methods for the view.

**Example ViewModel**

**`App/src/Auth/Views/Models/LoginViewModel.php`**

```php
<?php

namespace App\Auth\Views\Models;

use Helpers\Http\Flash;

readonly class LoginViewModel
{
    public function __construct(
        private Flash $flash
    ) {}

    public function getPageTitle(): string
    {
        return 'Login';
    }

    public function getFormActionUrl(): string
    {
        return url('auth/login');
    }

    public function hasError(string $field): bool
    {
        return $this->flash->hasInputError($field);
    }

    public function getErrorClass(string $field): string
    {
        return $this->hasError($field) ? 'is-invalid' : '';
    }
}
```

## CLI Generator

You can quickly generate a ViewModel using the `dock` CLI:

```bash
php dock view:create-model <ModelName> <ModuleName>
```

**Example:**

```bash
php dock view:create-model Login Auth
```

This will create `App/src/Auth/Views/Models/LoginViewModel.php`.

**Form ViewModel**

Use the `--form` (or `-f`) option to generate a ViewModel pre-populated with form helper methods:

```bash
php dock view:create-model Register Auth --form
```

The generated ViewModel will include methods for handling errors and persisting old input:

```php
// Checks if a field has an error
public function hasError(string $field): bool

// Returns 'is-invalid' class if field has error
public function getErrorClass(string $field): string

// Returns the flash/old value for a field
public function getFieldValue(string $field): string
```

## Usage

**In the Controller**

You can inject the ViewModel directly into your controller method:

```php
use App\Auth\Views\Models\LoginViewModel;

public function index(LoginViewModel $viewModel)
{
    // ViewModel is automatically resolved and injected
    return $this->asView('login', ['model' => $viewModel]);
}
```

Or manually resolve it from the container:

```php
// ...
    $viewModel = resolve(LoginViewModel::class);
// ...
```

**In the View**

Use the methods provided by the ViewModel.

```php
<!-- App/src/Auth/Views/Templates/login.php -->

<h1><?php echo $model->getPageTitle(); ?></h1>

<form action="<?php echo $model->getFormActionUrl(); ?>" method="POST">

    <div class="form-group">
        <label>Email</label>
        <input type="email" name="email" class="<?php echo $model->getErrorClass('email'); ?>">
    </div>

</form>
```

## Best Practices

- **Keep it Read-Only**: ViewModels should generally be immutable data holders.
- **Dependency Injection**: Use the constructor to inject Services, Flash helpers, or Request objects.
- **No Business Logic**: ViewModels should only contain logic related to _presentation_ (e.g., "is this button active?", "format this currency"), not core business rules.
- **One per View**: typically create one ViewModel per page (e.g., `ProfileViewModel`, `LoginViewModel`).
