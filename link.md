# Link

Link provides time-limited, scoped access tokens for resources. It enables secure sharing of files, media, support tickets, and invites with configurable expiration, usage limits, and recipient binding.

## Features

- **Timed Access**: Create links that expire after a configurable period.
- **Usage Limits**: Restrict links to single-use or a maximum number of accesses.
- **Scoped Permissions**: Define what actions the link allows (view, download, edit, join).
- **Recipient Binding**: Optionally bind links to specific emails, IPs, or users.
- **Signed URLs**: Cryptographically signed URLs for tamper-proof verification.
- **Revocation**: Manually revoke links before expiry.
- **Analytics**: Track access patterns, popular resources, and usage trends.

## Installation

Link is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Link --packages
```

This will automatically:

- Run database migrations for `link_*` tables.
- Register the `LinkServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/link.php`

```php
return [
    'default_expiry_hours' => 24,
    'max_expiry_days' => 365,
    'token_length' => 64,
    'signing_key' => null, // Falls back to app.key
    'retention_days' => 30,
    'rate_limit' => [
        'enabled' => true,
        'limit' => 100,
    ],
];
```

## Basic Usage

### Static Facade

Use the `Link` facade for fluent link creation:

```php
use Link\Link;

// Create a basic link to a document
$link = Link::make()
    ->for($document)
    ->validForHours(24)
    ->view()
    ->create();

// Get the shareable URL
$url = $link->signedUrl();
```

### Fluent API Examples

```php
use Link\Link;

// Single-use download link
$link = Link::make()
    ->for($media)
    ->download()
    ->singleUse()
    ->create();

// Week-long edit access
$link = Link::make()
    ->for($project)
    ->validForDays(7)
    ->edit()
    ->by($user->id)
    ->create();

// Invite link for team (integrates with Hub)
$link = Link::make()
    ->for($team)
    ->invite()        // Sets join scope + single use
    ->recipient($email)
    ->create();

// No expiration link with max 100 uses
$link = Link::make()
    ->for($resource)
    ->forever()
    ->maxUses(100)
    ->view()
    ->create();
```

### Validating Links

```php
use Link\Link;
use Link\Exceptions\LinkExpiredException;
use Link\Exceptions\LinkRevokedException;
use Link\Exceptions\LinkUsageExceededException;

try {
    $link = Link::validate($token);

    // Record usage fluently
    $link->recordUse($request->ip(), $request->userAgent());

    // Access the resource
    $resource = $link->linkable;

} catch (LinkExpiredException $e) {
    return response()->json(['error' => 'Link has expired'], 410);
} catch (LinkRevokedException $e) {
    return response()->json(['error' => 'Link was revoked'], 403);
} catch (LinkUsageExceededException $e) {
    return response()->json(['error' => 'Link usage limit reached'], 403);
}

// Or use safe validation (returns null on failure)
$link = Link::validateSafe($token);
if ($link === null) {
    throw new InvalidLinkException();
}
```

### Revoking Links

```php
use Link\Link;

// Revoke by token
Link::revoke($token);

// Revoke by reference ID
Link::revokeByRefid($refid);

// Or via model
$link = Link::find($refid);
$link->revoke();
```

## Model Integration

The `Link` model provides helper methods:

```php
$link = Link::find($refid);

// Status checks
$link->isValid();      // True if active
$link->isExpired();    // True if past expiry
$link->isRevoked();    // True if manually revoked
$link->isExhausted();  // True if max uses reached

// Scope checks (fluent)
$link->canView();      // True/false
$link->canDownload();  // True/false
$link->canEdit();      // True/false
$link->canJoin();      // For invites

// Get computed status
$link->getStatus();    // LinkStatus enum

// Remaining uses (null = unlimited)
$link->getRemainingUses();

// Access the linked resource
$resource = $link->linkable;
```

### Fluent Scopes

```php
use Link\Models\Link;

// Get all valid links
$active = Link::valid()->get();

// Get expired links
$expired = Link::expired()->get();

// Get links for a specific resource
$links = Link::forResource(Document::class, $docId)->get();

// Get links created by a user
$myLinks = Link::createdBy($userId)->get();
```

## Analytics

```php
use Link\Link;

$analytics = Link::analytics();

// Top accessed resources (last 30 days)
$topResources = $analytics->getTopResources(30);
// Returns: [['linkable_type' => 'App\Models\Document', 'linkable_id' => 5, 'access_count' => 142], ...]

// Daily usage trends (for line charts)
$trends = $analytics->getUsageTrends(30);
// Returns: [['date' => '2026-01-01', 'access_count' => 45], ['date' => '2026-01-02', 'access_count' => 67], ...]

// Expiration metrics (for pie charts)
$metrics = $analytics->getExpirationMetrics();
// Returns: ['total' => 150, 'active' => 95, 'expired' => 40, 'revoked' => 15, 'active_percentage' => 63.3]

// Most active creators (for bar charts)
$creators = $analytics->getTopCreators(10);
// Returns: [['created_by' => 1, 'link_count' => 42], ['created_by' => 3, 'link_count' => 28], ...]

// Scope distribution (for pie charts)
$scopes = $analytics->getScopeDistribution();
// Returns: ['view' => 120, 'download' => 85, 'edit' => 15, 'join' => 30]
```

## Package Integrations

### Vault Package

Share encrypted vault files with external clients securely.

```php
use Link\Link;
use Vault\Vault;

// Share a sensitive document with a client
$vaultFile = Vault::file($fileId);

$link = Link::make()
    ->for($vaultFile)
    ->validForHours(48)
    ->download()
    ->singleUse()
    ->recipient($clientEmail)
    ->by($user->id)
    ->create();

// Vault tracks the access via Link's usage events
```

### Media Package

Create shareable media links for galleries, portfolios, or client proofs.

```php
use Link\Link;
use Media\Media;

// Share a photo gallery for client review
$gallery = Media::gallery($galleryId);

$link = Link::make()
    ->for($gallery)
    ->validForDays(14)
    ->view()
    ->metadata(['purpose' => 'client_review', 'project' => $projectName])
    ->create();

// Client can view but not download without additional scope
```

### Support Package

Share tickets with external stakeholders or consultants.

```php
use Link\Link;
use Support\Support;

// Share a ticket with an external vendor for troubleshooting
$ticket = Support::findTicket($ticketId);

$link = Link::make()
    ->for($ticket)
    ->validForDays(5)
    ->view()
    ->recipient($vendorEmail)
    ->metadata(['reason' => 'External escalation'])
    ->by($user->id)
    ->create();
```

### Hub Package

Generate invite links for team threads and collaboration spaces.

```php
use Hub\Hub;
use Link\Link;

// Create thread and generate invite in one flow
$thread = Hub::thread()
    ->on($project)
    ->title('Project Discussion')
    ->by($user->id)
    ->create();

$invite = Link::make()
    ->for($thread)
    ->invite()
    ->recipient('contractor@external.com')
    ->metadata(['role' => 'guest', 'expires_access' => true])
    ->create();

// When invite is used, validate and add member:
$link = Link::validate($token);
if ($link->canJoin()) {
    $link->linkable
        ->addMember($newUserId)
        ->asGuest()
        ->add();

    $link->recordUse($request->ip(), $request->userAgent());
}
```

### Audit Package

Link automatically logs operations to Audit when the package is installed.

**Automatic logging** (zero config when Audit is installed):

- `link.created` - When a link is created via `Link::make()->create()`

**Manual logging** for additional events:

```php
use Audit\Audit;
use Link\Link;

// Validate and record access
$link = Link::validate($token);
$link->recordUse($request->ip(), $request->userAgent());

// Optionally add custom audit entry
Audit::make()
    ->event('link.accessed')
    ->on($link->linkable)
    ->data(['link_refid' => $link->refid, 'ip' => $request->ip()])
    ->log();
```

### Permit Package

Check permissions before creating links with specific scopes.

```php
use Link\Link;
use Permit\Permit;

// Only allow link creation if user has share permission
if (Permit::can(auth()->user(), 'share', $document)) {
    $link = Link::make()
        ->for($document)
        ->validForDays(7)
        ->download()
        ->by($user->id)
        ->create();
}
```

## Use Cases

#### Secure File Sharing

Share vault files with clients for a limited time.

```php
use Helpers\Data;
use Link\Link;
use Mail\Mail;

public function shareDocument(Document $document, string $recipientEmail)
{
    // Create a 48-hour download link
    $link = Link::make()
        ->for($document)
        ->validForHours(48)
        ->download()
        ->recipient($recipientEmail)
        ->by($user->id)
        ->create();

    // Send via Mail
    Mail::send(new DocumentSharedNotification(Data::make([
        'email' => $recipientEmail,
        'link' => $link->signedUrl(),
    ])));

    return $link->signedUrl();
}
```

#### Team Invites (with Hub)

The Link package integrates with Hub for team invitations.

```php
use Link\Link;

// Create an invite for a Hub thread
$invite = Link::make()
    ->for($thread)
    ->invite()
    ->recipient('new-member@company.com')
    ->metadata(['role' => 'member', 'team' => $team->name])
    ->create();
```

#### Support Ticket Sharing

Share a ticket temporarily with external stakeholders.

```php
use Link\Link;

$link = Link::make()
    ->for($ticket)
    ->validForDays(3)
    ->view()
    ->by($user->id)
    ->create();
```

#### SaaS: Client Portal Downloads

Allow clients to download invoices and reports via secure, time-limited links.

```php
use Helpers\Data;
use Link\Link;
use Mail\Mail;

class InvoiceService
{
    public function sendInvoiceLink(Invoice $invoice): void
    {
        $link = Link::make()
            ->for($invoice->pdfFile)
            ->validForDays(30)
            ->download()
            ->maxUses(5)
            ->recipient($invoice->client->email)
            ->metadata(['invoice_id' => $invoice->id])
            ->create();

        Mail::send(new InvoiceReadyNotification(Data::make([
            'client' => $invoice->client->name,
            'amount' => $invoice->formatted_total,
            'download_url' => $link->signedUrl(),
        ])));
    }
}
```

#### E-commerce: Order Tracking Links

Generate one-click tracking links for customers without login.

```php
use Link\Link;

class OrderService
{
    public function generateTrackingLink(Order $order): string
    {
        $link = Link::make()
            ->for($order)
            ->validForDays(90)
            ->view()
            ->recipient($order->customer_email)
            ->create();

        return $link->signedUrl();
    }
}
```

#### Healthcare: Secure Document Sharing

Share medical records with patients or other providers with strict controls.

```php
use Link\Link;

class MedicalRecordService
{
    public function shareWithPatient(MedicalRecord $record, string $patientEmail): string
    {
        // Single-use, short expiry, IP-bound for security
        $link = Link::make()
            ->for($record)
            ->validForHours(24)
            ->view()
            ->singleUse()
            ->recipient($patientEmail)
            ->metadata(['hipaa_consent' => true, 'shared_at' => DateTimeHelper::now()])
            ->by($user->id)
            ->create();

        return $link->signedUrl();
    }
}
```

#### Legal: Contract Review Links

Share contracts with clients for e-signature or review.

```php
use Link\Link;

class ContractService
{
    public function sendForReview(Contract $contract, array $reviewerEmails): array
    {
        $links = [];

        foreach ($reviewerEmails as $email) {
            $links[$email] = Link::make()
                ->for($contract)
                ->validForDays(7)
                ->view()
                ->edit()  // Allow annotations
                ->singleUse()
                ->recipient($email)
                ->metadata(['stage' => 'review', 'version' => $contract->version])
                ->create()
                ->signedUrl();
        }

        return $links;
    }
}
```

#### Marketing: Campaign Asset Distribution

Share marketing assets with partners or affiliates.

```php
use Link\Link;

class AssetDistributionService
{
    public function createPartnerAssetLink(Campaign $campaign, Partner $partner): string
    {
        $link = Link::make()
            ->for($campaign->assetBundle)
            ->validForDays(30)
            ->download()
            ->maxUses(10)
            ->metadata([
                'partner_id' => $partner->id,
                'campaign' => $campaign->name,
            ])
            ->create();

        return $link->signedUrl();
    }
}
```

#### HR: Onboarding Document Access

Provide new hires access to onboarding materials before their start date.

```php
use Link\Link;

class OnboardingService
{
    public function sendWelcomePackage(Employee $employee): void
    {
        $link = Link::make()
            ->for($employee->onboardingBundle)
            ->until($employee->start_date->addDays(30))
            ->view()
            ->download()
            ->recipient($employee->personal_email)
            ->by($user->id)
            ->create();

        // Send welcome email with link
    }
}
```

## Service API Reference

### Link (Facade)

| Method                          | Description                            |
| :------------------------------ | :------------------------------------- |
| `make()`                        | Returns a fluent `LinkBuilder`.        |
| `find(string $refid)`           | Finds a link by reference ID.          |
| `validate(string $token)`       | Validates token and returns link.      |
| `validateSafe(string $token)`   | Validates without throwing exceptions. |
| `isValid(string $token)`        | Checks if token is valid.              |
| `revoke(string $token)`         | Revokes a link by token.               |
| `revokeByRefid(string $refid)`  | Revokes a link by reference ID.        |
| `recordUsage($link, $metadata)` | Records an access event.               |
| `generateSignedUrl($link)`      | Generates a signed URL.                |
| `cleanup()`                     | Removes expired links past retention.  |
| `analytics()`                   | Returns the `LinkAnalytics` service.   |

### LinkBuilder (Fluent)

| Method                      | Description                      |
| :-------------------------- | :------------------------------- |
| `for(BaseModel $model)`     | Sets the resource to link to.    |
| `validForHours(int $h)`     | Sets expiry by hours.            |
| `validForDays(int $d)`      | Sets expiry by days.             |
| `validForMinutes(int $m)`   | Sets expiry by minutes.          |
| `until(Carbon $datetime)`   | Sets specific expiry datetime.   |
| `forever()`                 | No expiration.                   |
| `maxUses(int $count)`       | Sets maximum usage limit.        |
| `singleUse()`               | Shorthand for `maxUses(1)`.      |
| `scopes(array $scopes)`     | Sets allowed scopes.             |
| `view() / download()`       | Shorthand scope setters.         |
| `edit() / join() / share()` | Additional scope setters.        |
| `invite()`                  | Shorthand for join + single use. |
| `recipient(string $email)`  | Binds to email.                  |
| `recipientIp(string $ip)`   | Binds to IP address.             |
| `recipientUser(int $id)`    | Binds to user ID.                |
| `metadata(array $data)`     | Sets metadata.                   |
| `with(string $k, $v)`       | Adds metadata key.               |
| `by(int $userId)`           | Sets creator.                    |
| `create()`                  | Persists and returns the link.   |

### Link (Model)

| Method               | Type      | Description                        |
| :------------------- | :-------- | :--------------------------------- |
| `isValid()`          | `boolean` | True if link is usable.            |
| `isExpired()`        | `boolean` | True if past expiry.               |
| `isRevoked()`        | `boolean` | True if manually revoked.          |
| `isExhausted()`      | `boolean` | True if usage limit reached.       |
| `getStatus()`        | `Enum`    | Returns `LinkStatus` enum.         |
| `hasScope($scope)`   | `boolean` | Checks if scope is allowed.        |
| `revoke()`           | `void`    | Revokes the link.                  |
| `getRemainingUses()` | `?int`    | Remaining uses (null = unlimited). |
| `linkable`           | `Model`   | The linked resource (polymorphic). |

## Troubleshooting

| Error                        | Cause                      | Solution                          |
| :--------------------------- | :------------------------- | :-------------------------------- |
| `InvalidLinkException`       | Token not found.           | Check the token is correct.       |
| `LinkExpiredException`       | Link past expiry date.     | Create a new link.                |
| `LinkRevokedException`       | Link was manually revoked. | Create a new link.                |
| `LinkUsageExceededException` | Max uses reached.          | Create a new link with more uses. |

## Security Best Practices

- **Token Hashing**: Tokens are hashed before storage; plain tokens only exist temporarily.
- **Signed URLs**: Use signed URLs to prevent tampering.
- **Recipient Binding**: Bind sensitive links to specific recipients.
- **Short Expiry**: Use the shortest practical expiry for sensitive content.
- **Audit Trail**: Link usage is tracked for compliance.
