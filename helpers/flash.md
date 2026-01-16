# Flash Helper

The `Helpers\Http\Flash` class simplifies the management of one-time session messages, often used for user feedback after form submissions or redirects.

> Use the `flash()` global helper for quick access.

```php
// Set a success message
flash()->success('Profile updated!');

// Flash input and errors, then redirect back
flash()->withInput(request()->all(), $errors);

return redirect()->back();
```

## Basic Methods

#### success / error / info

```php
success(mixed $message): void

error(mixed $message): void

info(mixed $message): void
```

Convenience methods for setting the most frequent flash message types.

- **Use Case**: Informing the user that their password was changed (`success`) or that an API key is expiring (`info`).
- **Example**: `flash()->success('Operation complete!')`.

#### set

`set(string $type, mixed $message): void`

Sets a flash message for a custom type.

#### get

```php
get(string $key): mixed
```

Retrieves a flash message from the session and immediately **removes** it.

- **Use Case**: Consuming messages in a view template to display them once.
- **Example**: `<?php if ($msg = flash()->get('success')): ?> ... <?php endif; ?>`.

#### peek

```php
peek(string $key): mixed
```

Retrieves a flash message **without** removing it from the session.

- **Use Case**: Checking for a message early in the request cycle without preventing it from being displayed later in the view.

#### has

`has(string $key): bool`

Checks if a message of the specified type exists.

#### delete

`delete(string $key): void`

Manually removes a flash message.

## Input & Validation Errors

#### withInput

```php
withInput(array $formData, mixed $errors): void
```

Flashes the current form input data and validation errors to the session.

- **Use Case**: Preserving user-typed data and validation messages after a redirect-back from a failed form submission.

```php
flash()->withInput($request->post(), $validator->errors());
return redirect()->back();
```

#### old

`old(?string $key = null): mixed`

Retrieves old input data. If `$key` is null, returns the entire input array.

#### getInputError / peekInputError

`getInputError(string $field): mixed`

`peekInputError(string $field): mixed`

Retrieves (and potentially removes) or peeks at a validation error for a specific field.

#### hasInputError / hasInputErrors

`hasInputError(string $field): bool`

`hasInputErrors(): bool`

Checks for the existence of errors.

#### getInputErrors

`getInputErrors(): array`

Returns the entire array of validation errors.

## Convenience Checkers

#### hasSuccess / hasError / hasInfo

`hasSuccess(): bool`

`hasError(): bool`

`hasInfo(): bool`

Checks for standard message types.

#### getSuccess / getError / getInfo

`getSuccess(): mixed`

`getError(): mixed`

`getInfo(): mixed`

Retrieves and removes standard message types.

## Clearing Methods

#### clearSuccess / clearError / clearInfo

`clearSuccess(): void`

`clearError(): void`

`clearInfo(): void`

Clears standard flash types. Note that `clearError()` also clears validation errors.

## Global Helper

#### flash

`flash(?string $type = null, mixed $message = null): mixed`

- **No arguments**: Returns the `Flash` instance.
- **One argument**: Returns (and clears) the message for that `$type`.
- **Two arguments**: Sets the `$message` for that `$type`.

## Related

- [Session](session-helper.md) - Persistent data storage
- [Redirect](redirect-helper.md) - Handled via Response helper
- [Validation](validation-helper.md) - Validation logic and error generation
