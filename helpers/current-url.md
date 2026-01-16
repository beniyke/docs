# Current URL Helper

The `current_url()` global helper returns the full URL of the current request, including protocol, domain, and path.

#### Syntax

`current_url(): string`

#### Examples

```php
// If on https://example.com/products/electronics
echo current_url();
// Output: https://example.com/products/electronics

// Canonical Meta Tag
<link rel="canonical" href="<?= current_url() ?>" />

// Social Sharing
$shareUrl = "https://twitter.com/share?url=" . urlencode(current_url());
```

## Related

- [Request URI](request-uri-helper.md) - Path without domain
- [URL](url-helper.md) - URL generation
- [Request](request-helper.md) - Access request instance
