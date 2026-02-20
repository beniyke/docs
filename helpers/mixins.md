# Mixins Helper

The `Helpers\Mixins` trait allows you to bulk-register macros from another class or object. This is useful for grouping multiple related macros into a single "mixin" class.

## Usage

Use the trait in your class alongside `Helpers\Macroable`:

```php
use Helpers\Macroable;
use Helpers\Mixins;

class MyClass {
    use Macroable, Mixins;
}

// Define a mixin class
class MyMixins {
    public function sayHello() {
        return function($name) {
            return "Hello, " . $name;
        };
    }
}

// Register all methods from MyMixins as macros
MyClass::mixin(new MyMixins());

$instance = new MyClass();
echo $instance->sayHello('BenIyke'); // "Hello, BenIyke"
```

## Methods

### mixin

```php
static mixin(object|string $mixin, bool $replace = true): void
```

Registers all public and protected methods of the given class or object as macros on the target class.

- **Parameters**:
    - `$mixin`: An object instance or the fully qualified class name of the mixin.
    - `$replace`: Whether to replace existing macros with the same name. Defaults to `true`.
- **Requirements**: Each method in the mixin class should return a `Closure`. This closure will be registered as the macro.

## Related

- [Macroable](macroable.md) - The core trait for individual macro registration.
