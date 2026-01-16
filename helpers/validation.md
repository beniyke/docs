# Validation Helper

The `Validator` class provides a robust engine for verifying input data against a wide range of rules. It supports nested data, conditional logic, and custom error messages.

> Use the `validator()` global helper for quick access.

## Basic Usage

```php
$data = [
    'email' => 'user@example.com',
    'age'   => 25
];

$validator = validator();

// Set labels for human-readable error messages
$validator->parameters([
    'email' => 'Email Address',
    'age'   => 'User Age'
]);

// Define rules
$validator->rules([
    'email' => ['required' => true, 'type' => 'email'],
    'age'   => ['required' => true, 'type' => 'int', 'limit' => '18|100']
]);

if ($validator->validate($data)->has_error()) {
    $errors = $validator->errors();
} else {
    $validated = $validator->validated(); // Returns a Data object
}
```

## Validation Rules

### Data Types (`type`)

Validated via the `type` rule key.

| Type              | Description                                      |
| :---------------- | :----------------------------------------------- |
| `string`          | Must be a string                                 |
| `email`           | Valid email format                               |
| `int` / `integer` | Valid integer                                    |
| `number`          | Numeric value (float or int)                     |
| `boolean`         | Boolean value                                    |
| `url`             | Valid URL                                        |
| `phone`           | Standard phone number format                     |
| `alnum`           | Alphanumeric characters only                     |
| `alpha`           | Alphabetical characters only                     |
| `ipv4` / `ipv6`   | Valid IP address                                 |
| `creditcard`      | Validated via Luhn algorithm                     |
| `coordinate`      | Latitude,Longitude pair                          |
| `password`        | Complex check (uppercase, numeric, special, etc) |

### Constraints & Comparisons

| Rule           | Condition          | Description                                                     |
| :------------- | :----------------- | :-------------------------------------------------------------- |
| `required`     | `bool`             | Field must be present and not empty                             |
| `length`       | `int` or `min,max` | Exact length or range of characters                             |
| `minlength`    | `int`              | Minimum character count                                         |
| `maxlength`    | `int`              | Maximum character count                                         |
| `limit`        | `min\|max`         | Numerical range (e.g., `1\|100`)                                |
| `date`         | `string` (format)  | Must match format (e.g., `Y-m-d`). Supports ranges with `to`.   |
| `less_than`    | `string` (field)   | Must be less than another field's value                         |
| `greater_than` | `string` (field)   | Must be greater than another field's value                      |
| `same`         | `string` (field)   | Value must match another field exactly                          |
| `not_same`     | `string` (field)   | Value must not match another field                              |
| `not_contain`  | `string` (field)   | Must not contain any words present in the specified other field |

### File Validation

| Rule                | Condition        | Description                                 |
| :------------------ | :--------------- | :------------------------------------------ |
| `file`              | `bool`           | Checks if the field is a valid file upload  |
| `allowed_file_type` | `array`          | Allowed extensions (e.g., `['jpg', 'png']`) |
| `allowed_file_size` | `string`         | Max size with unit (e.g., `2mb`, `500kb`)   |
| `secure_file`       | `array` (config) | Deep validation (type, maxSize, etc)        |

### Database Checks

| Rule     | Condition                                 | Description                                                                                                                                              |
| :------- | :---------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exist`  | `table.column`                            | Ensures the value exists in the database                                                                                                                 |
| `unique` | `table.column:ignoreValue[,ignoreColumn]` | Ensures the value does **not** already exist in the database. Supports optional `:ignoreValue` (defaults to `id` column) or `:ignoreValue,ignoreColumn`. |

---

## Email Specialization (`strict`)

The `strict` rule enables deep email verification.

```php
$validator->rules([
    'email' => [
        'type'   => 'email',
        'strict' => 'disposable,mx,role,smtp'
    ]
]);
```

- `disposable`: Block temporary email providers.
- `mx`: Verify the domain has valid Mail Exchange records.
- `role`: Block role-based addresses (e.g., `admin@`).
- `smtp`: Verify the mailbox actually exists.

---

## Advanced Controls

### Conditional Validation

#### `optional(array $config)`

Apply rules only if a certain field matches a specific value.

```php
$validator->optional([
    'account_type' => [
        'value' => 'business',
        'rules' => ['reg_number' => ['required' => true]]
    ]
]);
```

#### `notempty(array $config)`

Apply rules only if the specified field is not empty.

```php
$validator->notempty([
    'referral_code' => ['rules' => ['campaign' => ['required' => true]]]
]);
```

### Data Shaping

#### `expected(array $keys)`

Whitelist the fields that should be returned by `validated()`. If a key is missing from input, validation fails.

#### `exclude(array $keys)`

Blacklist specific fields from the final output.

#### `modify(array $map)`

Rename fields in the final output: `['username' => 'login_id']`.

---

## Recursive & Nested Data

The validator handles dot-notation and wildcards for bulk validation of arrays.

```php
$validator->parameters([
    'items.*.quantity' => 'Quantity'
]);

$validator->rules([
    'items.*.quantity' => ['required' => true, 'type' => 'int']
]);
```

## Related

- [Data Helper](data-helper.md) - Handling the resulting object
- [Flash Helper](flash-helper.md) - Persistence between requests
- [HTML Helper](html-helper.md) - Layout & UI components
- [Encryption](encryption-helper.md) - Hashing sensitive data
