# Request URI Helper

The `request_uri()` global helper returns the current request path without the protocol, domain, or leading slash.

#### Syntax

`request_uri(): string`

#### Examples

```php
// If on https://example.com/admin/settings
echo request_uri();
// Output: admin/settings

// Active Navigation Highlighting
<nav>
    <a href="<?= url('dashboard') ?>" class="<?= request_uri() === 'dashboard' ? 'active' : '' ?>">Dashboard</a>
</nav>
```

## Related

- [Current URL](current-url-helper.md) - Full URL with domain
- [URL](url-helper.md) - URL generation
- [Request](request-helper.md) - Access request instance
