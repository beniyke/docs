# Bridge Authentication

Bridge provides a simple and robust way to issue API tokens for Single Page Applications (SPAs), mobile applications, and general third-party API clients.

## Introduction

Bridge provides **stateless, token-based API authentication** for applications that need to authenticate without session cookies. This is ideal for:

- **Cross-Domain SPAs**: When your frontend and backend are on different domains.
- **Mobile Applications**: iOS and Android apps requiring API access.
- **Third-Party Integrations**: External services accessing your API.

### Authentication Strategies

Bridge supports three primary strategies to fit different security needs:

- **Personal Access Tokens**: Traditional user-specific tokens with granular abilities (Permissions).
- **Dynamic API Keys**: Database-backed keys for third-party integrations (can be named/labeled).
- **Static Tokens**: Simple, config-based tokens for internal services (e.g., cron jobs).

All strategies utilize the `Authorization: Bearer {token}` header.

## Installation

Bridge is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install bridge --packages
```

This command will automatically:

- Publish the `bridge.php` global configuration.
- Publish and run migrations for `personal_access_token` and `api_key`.
- Register the `BridgeServiceProvider` and the global `BridgeAuthMiddleware`.

## Personal Access Tokens

This is the most common use case, where users generate tokens for their own account (e.g., "Mobile App Login").

### Setup the Model

Add the `HasApiTokens` trait and implement `TokenableInterface` in your `User` model:

```php
namespace App\Models;

use Bridge\Traits\HasApiTokens;
use Bridge\Contracts\TokenableInterface;
use Database\BaseModel;

class User extends BaseModel implements TokenableInterface
{
    use HasApiTokens;
}
```

### Creating Tokens

Use the `createToken` method on the user model. You can optionally specify a name and an array of abilities.

```php
$user = User::find(1);

// Create a token with 'read' and 'write' abilities
$token = $user->createToken('Mobile App', ['read', 'write']);

// createToken returns the plain-text token string
return ['token' => $token];
```

> The plain-text token is only available once. It is hashed in the database and cannot be retrieved later.

### Ability Checks

Bridge automatically attaches the current token to the authenticated model. Use `tokenCan()` to check permissions:

```php
public function update(Request $request)
{
    $user = $request->user(); // Returns the authenticated User model

    if (! $user->tokenCan('write')) {
        return response()->json(['error' => 'Forbidden'], 403);
    }

    // ...
}
```

### Revoking Tokens

```php
// Revoke a specific token by ID
$user->revokeToken($tokenId);

// Revoke ALL tokens for this user
$user->revokeAllTokens();
```

## Route-Based Strategies

For advanced scenarios, you can define strategies per-route in `App/Config/api.php`.

### Static Tokens (Internal Services)

Ideal for simple scripts or cron jobs.

```php
// App/Config/api.php
'api/v1/cron' => [
    'type' => 'static',
    'token' => env('INTERNAL_API_KEY'),
],
```

### Dynamic API Keys (Third-Party)

Database-backed keys that aren't necessarily tied to a "user" model in the traditional sense, but can be generated for integrations.

```php
// App/Config/api.php
'api/v1/external' => [
    'type' => 'dynamic',
],
```

Generate keys programmatically:

```php
use Bridge\Models\ApiKey;

$result = ApiKey::generate('Integration Name');
// Returns ['key' => '...', 'model' => $apiKeyInstance]
```

## Global Configuration

Modify `App/Config/bridge.php` to adjust global settings like expiration and pruning.

```php
return [
    // Token valid duration (null for never)
    'expiration' => 60 * 24 * 30, // 30 days

    // Automatic prefix for plain-text tokens
    'prefix' => 'bridge_',

    // Pruning of expired tokens older than X hours
    'prune' => [
        'enabled' => true,
        'hours' => 24,
    ],
];
```

## Security Best Practices

- **Keep Tokens Secret**: Treat API tokens like passwords. Never log them.
- **Use HTTPS**: Always serve your API over HTTPS to prevent token interception.
- **Use Scope/Abilities**: Limit tokens to only the permissions they need (Principle of Least Privilege).
- **Regular Pruning**: Keep your database clean by ensuring `prune.enabled` is `true`.
