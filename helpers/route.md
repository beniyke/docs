# Route Helper

The `route()` global helper retrieves routing information for the current request context or a specific URI.

#### Syntax

```php
route(?string $uri = null, bool $re_route = false): string
```

Resolves the logical route for a given URI or the current request.

- **Use Case**: Determining which controller/action is handling a request, or checking route permissions.
- **Example**: `route('blog/post/1')` might return `post/show`.

#### Examples

```php
// Get current route (e.g., 'home/index')
$current = route();

// Get route for specific URI
$path = route('users/profile');

// With re-routing
$adminPath = route('dashboard', true);
```

## Related

- [Request URI](request-uri-helper.md) - Path without domain
- [Current URL](current-url-helper.md) - Full URL with domain
- [URL](url-helper.md) - URL generation
