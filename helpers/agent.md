# UserAgent Helper

The `Helpers\Http\UserAgent` class provides a robust way to detect browser, platform, device, and robot information from HTTP user agent strings. It also includes utilities for IP address and referrer detection.

## How To Use

### Direct Instantiation

```php
use Helpers\Http\UserAgent;

$userAgent = new UserAgent($_SERVER);

$browser = $userAgent->browser();
$platform = $userAgent->platform();
```

### Using Dependency Injection

```php
use Helpers\Http\UserAgent;

class DeviceController
{
    public function __construct(
        private UserAgent $agent
    ) {}

    public function checkDevice()
    {
        if ($this->agent->isMobile()) {
            return response()->json(['device' => 'mobile']);
        }

        return response()->json(['device' => 'desktop']);
    }
}
```

### Helper Function (Convenience)

For quick access, use the `agent()` helper:

```php
$browser = agent()->browser();
$isMobile = agent()->isMobile();
```

> The `agent()` helper creates a new instance each time. For performance in loops or for deep testability, it is better to inject the `UserAgent` class or reuse a single instance.

## Browser Detection

#### browser

```php
browser(): string
```

Returns the name of the browser (e.g., 'Chrome', 'Firefox', 'Safari').

- **Use Case**: Tailoring UI features or workarounds for specific browser engines.

#### version

```php
version(): string
```

Retrieves the browser version number.

- **Use Case**: Checking for outdated browser versions to display a compatibility warning.

#### agentString

```php
agentString(): string
```

Returns the raw HTTP `User-Agent` header string.

### Supported Browsers

- Chrome
- Firefox
- Safari
- Internet Explorer (Trident/MSIE)
- Opera (OPR/Opera)

## Platform/OS Detection

#### platform

```php
platform(): string
```

Identifies the operating system and its version (e.g., 'Windows 11', 'Mac OS X', 'Android').

- **Use Case**: Offering different download links (e.g., `.exe` vs `.dmg`) based on the visitor's OS.

## Device Detection

#### device

```php
device(): string
```

Returns a normalized device type (e.g., 'iPhone', 'Android', 'PC').

#### mobile

```php
mobile(): ?string
```

Returns the specific mobile device name if the visitor is on mobile, otherwise `null`.

- **Use Case**: Identifying specific mobile hardware for advanced analytics.

## Mobile & Robot Detection

#### isMobile

```php
isMobile(): bool
```

Determines if the request originated from a mobile device (phone or tablet).

- **Use Case**: Switching between different layouts or simplifying the UI for touch interaction.

#### isRobot

```php
isRobot(): bool
```

Checks if the visitor is a known search engine crawler or bot.

- **Use Case**: Excluding crawlers from page view analytics or serving specialized SEO content.

#### robot

```php
robot(): ?string
```

Returns the name of the bot (e.g., 'Googlebot', 'Bingbot').

### Supported Bots

- Googlebot
- Bingbot
- Baiduspider
- YandexBot (Yandex)
- MSNBot

## Language Detection

```php
// Get accepted languages (parsed from HTTP_ACCEPT_LANGUAGE)
$languages = $agent->languages();
// Returns: ['en-us', 'en', 'fr']

// Check if specific language is accepted
if (in_array('fr', $agent->languages())) {
    // French is accepted
}
```

## Static Utilities

### IP Detection

The `ip()` method provides a reliable way to get the client's IP address, handling proxy headers automatically.

```php
use Helpers\Http\UserAgent;

$ip = UserAgent::ip();
// Or via server vars: UserAgent::ip($_SERVER);
```

### Referrer Detection

```php
$referrer = UserAgent::referrer();
// Returns the HTTP_REFERER if set, otherwise an empty string.
```

#### Examples

**Responsive Content Delivery**

```php
if ($agent->isMobile()) {
    return view('mobile/home', ['device' => $agent->device()]);
}

return view('desktop/home');
```

**Bot Handling**

```php
if ($agent->isRobot()) {
    // Serve simplified content for indexing
    return response('Static content for ' . $agent->robot());
}
```

**Platform-Specific Logic**

```php
$platform = $agent->platform();

if (str_contains($platform, 'Windows')) {
    $download = 'setup.exe';
} elseif (str_contains($platform, 'Mac')) {
    $download = 'installer.dmg';
}
```

## Best Practices

```php
// ✅ Recommended: Reuse instance
$agent = agent();

if ($agent->isMobile()) {
    $type = $agent->mobile();
}

// ❌ Avoid: Multiple instances for one request
if (agent()->isMobile()) {
    $type = agent()->mobile();
}
```

## Related

- [Request](request-helper.md) - Access request headers and data
- [Http Overview](http-helper.md) - Main HTTP documentation
