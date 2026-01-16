# URL Helper

The `url()` global helper generates a fully-qualified application URL with optional query parameters.

#### Syntax

```php
url(?string $path = null, array $query = []): string
```

Generates a fully qualified URL for the application.

- **Use Case**: Creating absolute links for emails, API responses, or cross-domain redirects.
- **Example**: `url('receipt', ['id' => 1234])` -> `https://example.com/receipt?id=1234`.

#### Examples

```php
// Application base URL
$base = url();

// Path relative to base
$url = url('profile/settings');

// URL with query string
$search = url('search', ['q' => 'anchor', 'page' => 1]);
// Result: https://domain.com/search?q=anchor&page=1

// Multi-value query params
$filter = url('products', ['category' => ['books', 'tech']]);
// Result: https://domain.com/products?category%5B0%5D=books&category%5B1%5D=tech
```

## Related

- [Route](route-helper.md) - Route information
- [Current URL](current-url-helper.md) - Full URL of current page
- [Request URI](request-uri-helper.md) - Path of current page
- [Redirect](redirect-helper.md) - Redirection responses
