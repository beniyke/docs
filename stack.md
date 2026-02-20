# Stack

Stack is a fluent form builder package for the Anchor Framework. It allows you to define complex forms programmatically, handle submissions with built-in validation, and track performance with robust analytics.

## Features

- **Fluent API**: Build forms and fields using a chainable, expressive syntax.
- **Dynamic Validation**: Automatically applies validation rules to submissions.
- **Media Integration**: Built-in support for file uploads (via `Media` package).
- **Hardened Analytics**: Track views, starts, completions, and errors per form.
- **Strict Isolation**: No direct model dependencies on other packages.

## Installation

Stack is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Stack --packages
```

This command will:

- Publish the `stack.php` configuration file.
- Run the migration for Stack tables.
- Register the `StackServiceProvider`.

## Basic Usage

### Creating a Form

You can use the `Stack` facade to define a new form programmatically.

```php
use Stack\Stack;

$form = Stack::form()
    ->title('Order Feedback')
    ->active()
    ->withField('customer_name', 'Your Name')
        ->type('text')
        ->required()
        ->add()
    ->withField('rating', 'Rating')
        ->type('select')
        ->option('Excellent', '5')
        ->option('Good', '4')
        ->option('Average', '3')
        ->option('Poor', '1')
        ->required()
        ->add()
    ->withField('comments', 'Additional Comments')
        ->type('textarea')
        ->add()
    ->create();
```

### Handling Submissions

To submit data to a form, use the `submit` method. It returns a `Submission` model.

```php
use Stack\Stack;
use Exceptions\ValidationException;

try {
    $submission = Stack::submit($form, [
        'customer_name' => 'John Doe',
        'rating' => '5',
        'comments' => 'Great service!'
    ]);

    echo "Form submitted! Ref ID: " . $submission->refid;
} catch (ValidationException $e) {
    // Handle validation errors...
    print_r($e->getErrors());
}
```

### Tracking Events

Stack handles view and error tracking automatically if used via the manager, but you can also record manual events.

```php
use Stack\Stack;

Stack::recordEvent($form, 'view');
```

## Analytics

Access form-level analytics through the `Stack::analytics()` service.

```php
$metrics = Stack::analytics()->getConversionMetrics($form);
// Returns: ['views' => 100, 'submissions' => 10, 'conversion_rate' => 10.0]

echo "Views: " . $metrics['views'];
echo "Submissions: " . $metrics['submissions'];
echo "Conversion Rate: " . $metrics['conversion_rate'] . "%";
```

## Integration with Other Packages

Stack integrates gracefully with several Anchor packages:

- **Audit**: Logs form creation, deletions, and submissions (if installed).
- **Media**: Handles file uploads by linking `media_id` to submission values (if installed).
- **Link**: You can generate temporary, signed access links for private forms.

> Stack uses **Defensive Integration**. If a package like `Audit` or `Media` is missing, Stack will continue to function without errors, skipping the extra logging or upload features.
