# Support

The Support package offers a helpdesk and ticketing system, including SLA tracking, agent assignment, multi-channel support, and comprehensive analytics. It ensures efficient issue resolution through automated workflows and data-driven insights.

## Features

- **Fluent Ticketing**: Chainable API for creating and managing tickets.
- **SLA Management**: Priority-based response and resolution time tracking.
- **Agent Workflows**: Seamless ticket assignment and status transitions.
- **Internal Notes**: Support for private agent-only communication.
- **Auto-Closure**: Automated cleanup of long-resolved tickets.
- **Analytics**: Dashboard-ready reporting on agent performance and resolution trends.
- **Rate Limiting**: Protects your support infrastructure from abuse.

## Installation

Support is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Support --packages
```

This will automatically:

- Run the migration for Support tables.
- Register the `SupportServiceProvider`.
- Publish the configuration file.

### Configuration

Configuration file: `App/Config/support.php`

```php
return [
    // Default priority for new tickets
    'default_priority' => 'medium',

    // SLA hours based on priority
    'priorities' => [
        'low' => ['sla_hours' => 72],
        'medium' => ['sla_hours' => 24],
        'high' => ['sla_hours' => 8],
        'urgent' => ['sla_hours' => 2],
    ],

    // Auto-close resolved tickets after X days
    'auto_close_days' => 7,

    // Rate limiting settings
    'rate_limit' => [
        'enabled' => true,
        'limit' => 5, // tickets per hour
    ],
];
```

## Basic Usage

### Static Facade

Use the `Support` facade for fluent ticket creation and management:

```php
use Support\Support;

// Create a new ticket (Fluent)
$ticket = Support::make()
    ->for($user->id)
    ->category($categoryId)
    ->subject('Issue with order #123')
    ->description('The package arrived damaged...')
    ->high() // Set priority
    ->create();

// Reply to a ticket
Support::reply($ticket, $agentId, 'We are looking into this.');

// Reply with an internal note
Support::reply($ticket, $agentId, 'Checking shipping logs...', true);

// Resolve and Close
Support::resolve($ticket);
Support::close($ticket);

/**
 * Categories
 */

// Get all active categories (Fluent)
$active = Support::categories();

// Get inactive categories
$inactive = Support::categories(false);

// Create a new category using the fluent builder
$category = Support::configCategory()
    ->name('Technical Support')
    ->description('Help with software installation and bugs.')
    ->order(1)
    ->create();
```

## Ticket Management

### For Agents

Find tickets that need attention or have been assigned:

```php
// Get all unassigned open tickets
$unassigned = Support::getUnassignedTickets();

// Get tickets assigned to a specific agent
$myTickets = Support::getAgentTickets($agentId);

// Get tickets that have breached SLA
$overdue = Support::getSlaBreachedTickets();
```

### For Users

Retrieving a user's ticket history:

```php
// Get all tickets for a specific user
$history = Support::getUserTickets($userId);
```

## Model Integration

### Ticket Model

The `Ticket` model provides helper methods for state management:

```php
if ($ticket->isOpen()) {
    $ticket->assignTo($agentId);
}

if ($ticket->isSlaBreached()) {
    // Notify supervisors...
}

$ticket->reopen();
```

## Use Case Walkthrough

### Customer Support Workflow

#### Customer Creates Ticket

```php
// SupportController.php
public function store(): Response
{
    $ticket = Support::make()
        ->for($this->auth->user()->id)
        ->subject($this->request->post('subject'))
        ->description($this->request->post('message'))
        ->medium()
        ->create();

    return $this->response->json([
        'ticket_id' => $ticket->refid,
        'status' => 'Ticket created successfully'
    ]);
}
```

#### Agent Responds & Assigns

```php
// AgentController.php
public function respond(string $refid): Response
{
    $ticket = Support::find($refid);
    $agentId = $this->auth->user()->id;

    // Assign to self and respond
    $ticket->assignTo($agentId);
    Support::reply($ticket, $agentId, $this->request->post('reply'));

    return $this->response->json(['status' => 'Reply sent']);
}
```

## Advanced Features

### SLA Tracking

Tickets automatically calculate a `sla_due_at` timestamp based on their priority. You can easily find tickets that need immediate attention:

```php
// Get all tickets that breached SLA
$overdue = Support::getSlaBreachedTickets();

// Get unassigned open tickets
$unassigned = Support::getUnassignedTickets();
```

### Rate Limiting

The Support package includes a built-in rate limiter to prevent spamming.

```php
$limiter = Support::rateLimiter();

// Check if user can create a ticket (default limit is 5 per hour)
if ($limiter->canCreateTicket($userId)) {
    // Proceed...
} else {
    $wait = $limiter->getTimeUntilNextTicket($userId);
    echo "Please wait {$wait} seconds.";
}

// Check if user is spamming replies
if ($limiter->canReply($userId, $ticketId)) {
    // Proceed...
}
```

### Analytics & Reporting

The `SupportAnalytics` service provides high-resolution data for your support dashboards. It uses optimized SQL aggregations to handle large datasets efficiently.

```php
use Support\Analytics;

// 1. Dashboard Overview
$dashboard = Analytics::getDashboard();
// Returns: total_open, total_resolved, avg_first_response, success_rate

// 2. SLA Compliance Trends
$sla = Analytics::getSlaMetrics();
// Returns: { breached: 12, compliance_rate: 98.2, at_risk: 5 }

// 3. Daily Ticket Volume (Chart Data)
$volume = Analytics::getDailyTrends(30);
// Returns array of { date: '2026-01-01', created: 45, resolved: 38 }

// 4. Agent League Table
$performance = Analytics::getAgentPerformance(10);
// Returns: { agent_name: 'John Doe', tickets_resolved: 150, avg_rating: 4.8 }
```

## Use Cases

### Enterprise SLA Escalation

For high-value accounts, any ticket marked as "Urgent" must be escalated to a senior engineer if not responded to within 60 minutes.

#### Implementation

```php
use Support\Support;
use Support\Enums\TicketPriority;
use Support\Events\TicketCreated;
use Mail\Mail;

// SupportServiceProvider or dedicated Listener
public function handleTicketEscalation(TicketCreated $event)
{
    $ticket = $event->ticket;

    // Use the fluent model helper for better DX
    if ($ticket->isUrgent()) {
        // Dispatch urgent notification to the on-call team
        Mail::send(new UrgentTicketAlert(Data::make([
            'recipient_email' => 'oncall@example.com',
            'recipient_name' => 'On-Call Team',
            'subject' => $ticket->subject,
            'refid' => $ticket->refid,
            'customer_name' => $ticket->user->name,
            'description' => $ticket->description,
            'manage_url' => "admin/support/tickets/{$ticket->refid}"
        ])));

        // Audit Integration: Log the escalation event
        Audit::make()
            ->event('sla.urgent_escalation')
            ->on($ticket)
            ->with('expected_resolution', $ticket->sla_due_at->toDateTimeString())
            ->log();
    }
}
```

### Sample Data (JSON)

State of an urgent ticket in the system:

```json
{
  "refid": "sup_secure_12345",
  "subject": "DB Cluster Connectivity Issues",
  "priority": "urgent",
  "status": "open",
  "sla_due_at": "2026-01-02 11:30:00",
  "metadata": {
    "cluster_id": "prod-east-1",
    "severity": "P0"
  }
}
```

### Email Notifications

The Support package provides built-in email alerts for key events. These notifications handle their own delivery logic using the system's `Mail` infrastructure.

```php
use Support\Notifications\TicketRepliedNotification;
use Mail\Mail;

// Notifying a user of an agent reply
public function handleReply($ticket, $reply)
{
    // Send email if the reply isn't internal
    if (!$reply->is_internal) {
        Mail::send(new TicketRepliedNotification(Data::make([
            'name' => $ticket->user->name,
            'email' => $ticket->user->email,
            'subject' => $ticket->subject,
            'refid' => $ticket->refid,
            'reply_message' => $reply->message,
            'ticket_url' => "support/tickets/{$ticket->refid}"
        ])));
    }
}
```

### Audit Package (Traceability)

Every action is logged. You can query the history of a specific ticket easily:

```php
use Audit\Audit;

$history = Audit::history()
    ->on($ticket)
    ->latest()
    ->get();
```

### Vault Package (Secure Storage)

Tickets use `Vault` for storing sensitive attachments like log files or screenshots.

```php
use Media\Media;
use Vault\Vault;

public function attachLogs($ticket, $file)
{
    // 1. Upload media normally
    $media = Media::upload($file, ['collection' => 'support_logs']);

    // 2. Track in Vault to enforce user storage quotas
    Vault::forAccount($ticket->user_id)
        ->trackUpload($media->getPath(), $media->size);

    $ticket->addMedia($media);
}
```

## Troubleshooting

| Error/Log                     | Cause                                     | Solution                                     |
| :---------------------------- | :---------------------------------------- | :------------------------------------------- |
| `SupportRateLimitException`   | Too many tickets created by a single user | Increase limit in config or wait             |
| "Reference ID already exists" | Rare collision in secure random generator | Retry the operation                          |
| "Category not found"          | Attempting to use an invalid category_id  | Verify category exists in `support_category` |

## Service API Reference

### Support (Facade)

| Method                                     | Description                                     |
| :----------------------------------------- | :---------------------------------------------- |
| `make()`                                   | Starts a fluent `TicketBuilder`.                |
| `createTicket(array $data)`                | Directly creates a ticket from an array.        |
| `reply($ticket, $userId, $msg, $internal)` | Adds a reply (or internal note).                |
| `assign($ticket, $agentId)`                | Assigns a ticket to a specific agent.           |
| `resolve($ticket)`                         | Marks a ticket as resolved.                     |
| `close($ticket)`                           | Marks a ticket as closed.                       |
| `find(string $refid)`                      | Finds a ticket by its unique reference ID.      |
| `categories()`                             | Returns list of active `TicketCategory` models. |
| `configCategory()`                         | Returns a fluent `CategoryBuilder` instance.    |
| `getUserTickets(int $userId)`              | Returns ticket history for a specific user.     |
| `getAgentTickets(int $agentId)`            | Returns open tickets assigned to an agent.      |
| `getUnassignedTickets()`                   | Returns all open tickets without an agent.      |
| `getSlaBreachedTickets()`                  | Returns open tickets that missed their SLA.     |
| `analytics()`                              | Returns the `SupportAnalytics` service.         |
| `rateLimiter()`                            | Returns the `SupportRateLimiter` service.       |

### TicketCategory (Model)

| Attribute       | Type      | Description                            |
| :-------------- | :-------- | :------------------------------------- |
| `name`          | `string`  | Display name of the category.          |
| `slug`          | `string`  | Unique URL-friendly identifier.        |
| `description`   | `string`  | Optional detailed description.         |
| `is_active`     | `boolean` | Whether category is visible to users.  |
| `display_order` | `integer` | Used for sorting categories in the UI. |

### CategoryBuilder (Fluent)

| Method                 | Description                                   |
| :--------------------- | :-------------------------------------------- |
| `name(string $name)`   | Sets display name and generates default slug. |
| `slug(string $slug)`   | Manually overrides the URL-friendly slug.     |
| `description($text)`   | Adds a detailed description.                  |
| `active(bool $status)` | Sets the initial visibility status.           |
| `inactive()`           | Shorthand for `active(false)`.                |
| `order(int $order)`    | Sets the sorting priority.                    |
| `create()`             | Finalizes and persists the category model.    |

### Analytics (Facade)

| Method                      | Description                                                  |
| :-------------------------- | :----------------------------------------------------------- |
| `getDashboard()`            | Returns complete dashboard overview metrics.                 |
| `getSlaMetrics()`           | Returns SLA compliance, breach, and at-risk counts.          |
| `getAgentPerformance($lim)` | Returns top performing agents.                               |
| `getResponseTimeMetrics()`  | Returns average first response and resolution times (hours). |
| `getDailyTrends($days)`     | Returns daily ticket creation and resolution trends.         |

### TicketBuilder (Fluent)

| Method               | Description                                   |
| :------------------- | :-------------------------------------------- |
| `for(int $id)`       | Sets the user who owns the ticket.            |
| `category(int $id)`  | Assigns the ticket to a category.             |
| `subject(string $s)` | Sets the ticket subject.                      |
| `description($d)`    | Sets the detailed description.                |
| `low() / urgent()`   | Quick methods to set priority.                |
| `priority($p)`       | Sets priority (string or Enum).               |
| `metadata(array $m)` | Attaches additional data.                     |
| `create()`           | Persists the ticket and returns the instance. |

## Console Commands

| Command                 | Description                                    |
| :---------------------- | :--------------------------------------------- |
| `support:auto-close`    | Automatically closes tickets resolved long ago |
| `support:stats`         | Quick overview of today's support metrics      |
| `support:fix-relations` | Audits and fixes orphaned ticket data          |

## Security Best Practices

- **Rate Limiting**: Always use `Support::rateLimiter()` in public controllers to prevent ticket spam and DoS attacks.
- **Internal Notes**: Ensure that `is_internal` replies are never exposed to the customer in the frontend.
- **SLA Integrity**: SLAs are calculated automatically on creation. Avoid manual `sla_due_at` updates to maintain reporting accuracy.
- **Data Privacy**: Use the `for($userId)` builder method to ensure tickets are correctly assigned to the authenticated user.
