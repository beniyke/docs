# Macroable Helper

The `Helpers\Macroable` trait enables classes to be dynamically extended with new methods at runtime. This "mixin" pattern is commonly used for adding custom functionality to framework core classes without modification or inheritance.

## Usage

Simply use the trait in your class:

```php
use Helpers\Macroable;

class MyClass {
    use Macroable;
}

// Add a macro
MyClass::macro('sayHi', function($name) {
    return "Hi, " . $name;
});

// Call it
echo (new MyClass())->sayHi('John'); // "Hi, John"
echo MyClass::sayHi('Jane');        // "Hi, Jane" (Static call works too)
```

## Methods

#### macro

```php
static macro(string $name, callable $macro): void
```

Registers a custom method that can be called on the class or its instances.

- **Use Case**: Extending a core framework class with a utility method specific to your project.
- **Example**: Adding a `toCSV` method to a `Collection` class.

#### hasMacro

```php
static hasMacro(string $name): bool
```

Checks if a specific macro has been registered.

- **Use Case**: Conditional logic that relies on a plugin or package having registered its macros.

#### flushMacros

```php
static flushMacros(): void
```

Removes all registered macros for the class.

- **Use Case**: Clearing state in a testing environment to ensure isolation.

## Internal Mechanics

The trait uses PHP's `__call` and `__callStatic` magic methods to intercept calls to non-existent methods and resolve them against the registered macros.

## Related

- [Data](data-helper.md) - Often uses macros for custom data transformations
- [Capsule](capsule-helper.md) - Supports custom getters and logic extensions
