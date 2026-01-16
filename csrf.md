# CSRF Protection

Cross-Site Request Forgery (CSRF) is a malicious exploit where unauthorized commands are performed on behalf of an authenticated user. Anchor automatically protects your application from CSRF attacks.

## How It Works

Anchor generates a unique CSRF token for each active user session. This token is verified on every state-changing request (POST, PUT, PATCH, DELETE) to ensure the authenticated user is actually making the request.

## Configuration

CSRF protection is configured in `App/Config/default.php`:

```php
'csrf' => [
    'enable' => true,
    'persist' => false,  // Regenerate token after each request
    'origin_check' => true,  // Verify IP and user agent
    'honeypot' => true,  // Enable honeypot field
    'routes' => [
        'exclude' => [],  // Routes to exclude from CSRF protection
    ],
],
```

## Using CSRF Tokens in Forms

### In Views (Recommended)

Use `$this->csrf()` in view templates:

```php
<form method="POST" action="<?php echo url('account/update'); ?>">
    <?php echo $this->csrf(); ?>

    <input type="text" name="name">
    <button type="submit">Update</button>
</form>
```

### What It Generates

The `$this->csrf()` method generates:

```html
<input type="hidden" name="csrf_token" value="base64_encoded_token_here" />
```

If honeypot is enabled, it also generates:

```html
<input type="hidden" name="top_yenoh" value="" />
```

### Important Form Fields Helper

For forms with method spoofing (PUT, PATCH, DELETE):

```php
<form method="POST" action="<?php echo url('posts/delete'); ?>">
    <?php echo $this->importantFormFields('DELETE'); ?>

    <button type="submit">Delete Post</button>
</form>
```

This generates:

- CSRF token
- Method field (`_method`)
- Callback route field
- Honeypot (if enabled)

## Getting the CSRF Token

**In PHP**

Use the `csrf_token()` helper:

```php
$token = csrf_token();
```

**In Views**

```php
<meta name="csrf-token" content="<?php echo csrf_token(); ?>">
```

**In JavaScript (AJAX)**

```javascript
// Get token from meta tag
const token = document.querySelector('meta[name="csrf-token"]').content;

// Include in AJAX requests
fetch("/api/endpoint", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-CSRF-Token": token,
  },
  body: JSON.stringify(data),
});
```

Or include in form data:

```javascript
const formData = new FormData();
formData.append("csrf_token", token);
formData.append("name", "John");

fetch("/api/endpoint", {
  method: "POST",
  body: formData,
});
```

## How Validation Works

1. **Token Generation**: Created when session starts, stored in session
2. **Token Embedding**: Added to forms via `$this->csrf()`
3. **Token Submission**: Sent with POST/PUT/PATCH/DELETE requests
4. **Token Validation**: Middleware validates token matches session
5. **Token Regeneration**: Optionally regenerated after each request (if `persist` is false)

## Token Expiry

CSRF tokens expire based on session timeout:

```php
// In App/Config/session.php
'timeout' => 3600,  // 1 hour
```

## Origin Checking

When `origin_check` is enabled, the token is bound to:

- User's IP address
- User's user agent

This provides additional security but may cause issues if users switch networks or browsers.

## Honeypot Protection

The honeypot is a hidden field that should remain empty. Bots typically fill all fields, triggering detection:

```php
// In request validation
if ($this->request->isBot()) {
    // Request blocked - honeypot was filled
}
```

## Excluding Routes from CSRF Protection

Configure excluded routes in `App/Config/default.php`:

```php
'csrf' => [
    'routes' => [
        'exclude' => [
            'api/*',
            'webhooks/*',
        ],
    ],
],
```

## Manual Validation

Check CSRF token manually:

```php
use Helpers\Http\Request;

public function store(Request $request)
{
    if (!$request->isSecurityValid()) {
        // CSRF validation failed
        return $this->response->status(403)->body('Invalid CSRF token');
    }

    // Process request
}
```

## Examples

### Standard Form

```php
<form method="POST" action="<?php echo url('account/profile'); ?>">
    <?php echo $this->csrf(); ?>

    <div>
        <label>Name</label>
        <input type="text" name="name" value="<?php echo $this->escape($user->name); ?>">
    </div>

    <div>
        <label>Email</label>
        <input type="email" name="email" value="<?php echo $this->escape($user->email); ?>">
    </div>

    <button type="submit">Update Profile</button>
</form>
```

### Form with Method Spoofing

```php
<form method="POST" action="<?php echo url('posts/' . $post->id); ?>">
    <?php echo $this->importantFormFields('PUT'); ?>

    <input type="text" name="title" value="<?php echo $this->escape($post->title); ?>">
    <textarea name="content"><?php echo $this->escape($post->content); ?></textarea>

    <button type="submit">Update Post</button>
</form>
```

### Delete Form

```php
<form method="POST" action="<?php echo url('posts/' . $post->id); ?>"
      onsubmit="return confirm('Are you sure?')">
    <?php echo $this->importantFormFields('DELETE'); ?>
    <button type="submit">Delete</button>
</form>
```

### AJAX Request

```javascript
// Set up CSRF token for all AJAX requests
const csrfToken = document.querySelector('meta[name="csrf-token"]').content;

// Using fetch
fetch("/api/posts", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    csrf_token: csrfToken,
    title: "My Post",
    content: "Post content",
  }),
})
  .then((response) => response.json())
  .then((data) => console.log(data));

// Using XMLHttpRequest
const xhr = new XMLHttpRequest();
xhr.open("POST", "/api/posts");
xhr.setRequestHeader("Content-Type", "application/json");
xhr.send(
  JSON.stringify({
    csrf_token: csrfToken,
    title: "My Post",
  })
);
```

## Security Best Practices

1. **Always use CSRF protection**: Don't disable it unless absolutely necessary
2. **Use HTTPS**: CSRF tokens can be intercepted over HTTP
3. **Keep tokens secret**: Never expose tokens in URLs or logs
4. **Regenerate tokens**: Use `persist: false` for maximum security
5. **Validate on server**: Never rely on client-side validation alone
6. **Use SameSite cookies**: Configure session cookies with SameSite attribute
7. **Enable origin checking**: For additional security (if users don't switch networks)

## Troubleshooting

### "Invalid CSRF Token" Error

**Causes**:

- Token expired (session timeout)
- Token mismatch (form cached, session regenerated)
- Origin check failed (IP or user agent changed)
- Missing token in request

**Solutions**:

- Increase session timeout
- Disable origin checking if users switch networks
- Ensure forms include `$this->csrf()`
- Check browser is sending cookies

### AJAX Requests Failing

**Solution**: Include CSRF token in request:

```javascript
// In request body
body: JSON.stringify({
    csrf_token: csrfToken,
    // other data
})

// Or in headers
headers: {
    'X-CSRF-Token': csrfToken
}
```

### Token Not Persisting

**Cause**: Session not starting properly

**Solution**: Ensure `SessionMiddleware` is registered and session configuration is correct.

## Available Methods

**In Views**

- `$this->csrf()` - Generate CSRF token field
- `$this->importantFormFields($method)` - Generate CSRF + method + callback fields
- `$this->hidden($name, $value)` - Generate hidden field
- `$this->method($verb)` - Generate method spoofing field

**In PHP**

- `csrf_token()` - Get current CSRF token
- `$request->getCsrfToken()` - Get or generate CSRF token
- `$request->isSecurityValid()` - Validate CSRF token and honeypot

## Token Structure

CSRF tokens are base64-encoded and contain:

- Timestamp (for expiry)
- IP + User Agent hash (if origin checking enabled)
- Random bytes (for uniqueness)

Example: `base64:dGltZXN0YW1wX2hhc2hfcmFuZG9tYnl0ZXM=`
