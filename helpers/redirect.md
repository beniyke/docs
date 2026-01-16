# Redirect Helper

The `redirect()` global helper and the `Response` class provide a suite of methods for handling HTTP redirections.

## Global redirect()

Create a redirect response with optional query parameters.

#### Syntax

```php
redirect(string $url = '', array $params = [], bool $internal = true): Response
```

- `$url`: The target URL or path.
- `$params`: Query parameters to append.
- `$internal`: If `true` (default), the base URL is prepended to the path.
- **Use Case**: Simple redirection after a successful action.
- **Example**: `return redirect('profile/settings', ['tab' => 'security'])`.

#### Examples

```php
// Internal path
return redirect('dashboard');

// Internal path with query params
return redirect('search', ['q' => 'anchor']);

// External URL
return redirect('https://example.com', [], false);
```

## Response Redirection

The `Response` class provides methods for more specific redirect types.

#### back

```php
back(): self
```

Redirects the user back to the previous URL (`HTTP_REFERER`). If no referer is found, it defaults to the site root.

- **Use Case**: Returning the user to a form after validation fails.

#### seeOther

```php
seeOther(string $url): self
```

Performs an HTTP 303 redirect. This is a best practice for redirecting after a POST request to ensure that refreshing the page doesn't re-submit the form.

- **Example**: `return response()->seeOther('success-page')`.

#### Examples

```php
// Basic back
return response()->back();

// With flash data (use flash helper)
flash('success', 'Profile updated!');

return redirect('profile');

// Permanent redirect
return response()->permanentRedirect('new-url');
```

## Related

- [Response](response-helper.md) - Base response management
- [URL](url-helper.md) - URL generation utilities
- [Route](route-helper.md) - Named route resolution
- [Flash](flash-helper.md) - Flash message management
