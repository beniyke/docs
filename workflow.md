# Workflow

The Workflow provides a durable, replay-based execution engine for long-running processes. It allows you to write complex, multi-step logic as code that can withstand system restarts, network failures, and long delays.

### Why Workflows?

- **Durability**: If the server crashes mid-process, the workflow resumes exactly where it left off.
- **Observability**: Every step is recorded in the `workflow_history` table.
- **Reliability**: Built-in retries and timeouts for every activity.
- **Simplicity**: Write complex asynchronous code in a linear, synchronous style.

## Installation

Workflow is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Workflow --packages
```

This command automatically:

- Publish the `workflow_history` migration.
- **Automatically runs the migration** to create history tracking tables.
- Register the `WorkflowServiceProvider`.
- Set up core bindings for the Engine and Runner.

### CLI Commands

The system provides generators to quickly scaffold workflows and activities.

> The system automatically appends the `Workflow` or `Activity` suffix to the class name if you omit it. For example, `workflow:create Onboarding` generates `OnboardingWorkflow`.

### Create Workflow

Generates a new workflow class.

```bash
# Shared workflow (App/Workflows)
php dock workflow:create Name

# Modular workflow (App/src/Module/Workflows)
php dock workflow:create Name Module
```

### Create Activity

Generates a new activity class.

```bash
# Shared activity (App/Activities)
php dock activity:create Name

# Modular activity (App/src/Module/Activities)
php dock activity:create Name Module
```

### Delete Workflow

Deletes an existing workflow class.

```bash
# Shared workflow
php dock workflow:delete Name

# Modular workflow
php dock workflow:delete Name Module
```

### Delete Activity

Deletes an existing activity class.

```bash
# Shared activity
php dock activity:delete Name

# Modular activity
php dock activity:delete Name Module
```

## Core Concepts

| Concept      | Description                                                           |
| :----------- | :-------------------------------------------------------------------- |
| **Workflow** | The orchestrator. Defines the business logic and sequence of steps.   |
| **Activity** | A single, idempotent unit of work (e.g., ChargeCard, SendEmail).      |
| **Command**  | An instruction yielded by a workflow (Activity, Timer, SideEffect).   |
| **History**  | The source of truth for replay. Records every "Event" in an instance. |

## Workflow Organization

Anchor supports a flexible architecture for organizing workflows. A **hybrid approach** based on your application's modularity.

### Modular (Domain-Specific)

For workflows that belong to a specific business domain, place them within the module's directory. This keeps domain logic encapsulated.

**Example Structure:**

- `App/src/Account/Workflows/ResetPasswordWorkflow.php`
- `App/src/Account/Activities/GenerateResetToken.php`

### Shared (Cross-Cutting/Orchestration)

For workflows that coordinate multiple modules or perform generic system tasks, use a central location.

**Example Structure:**

- `App/Workflows/SystemCleanupWorkflow.php`
- `App/Activities/NotifyAdmin.php`

> Use **Modular** for 90% of your business logic. Use **Shared** only when a workflow acts as a "bridge" between two or more disconnected modules.

## Creating a Workflow

Implement the `Workflow\Contracts\Workflow` interface. Use `yield` to trigger activities or commands.

```php
namespace App\Account\Workflows;

use Generator;
use Workflow\Contracts\Workflow;
use App\Account\Activities\CreateUserRecord;
use App\Account\Activities\SendWelcomeEmail;

class UserOnboardingWorkflow implements Workflow
{
    /**
     * The main execution logic.
     */
    public function execute(array $input): Generator
    {
        // Step 1: Create the user (returns the result of the activity)
        $userId = yield new CreateUserRecord($input);

        // Step 2: Send welcome email
        yield new SendWelcomeEmail(['user_id' => $userId]);

        return "Onboarding complete for user: $userId";
    }

    /**
     * Handle external signals (e.g., manual approval, cancellation).
     */
    public function handleSignal(string $signalName, array $payload): void
    {
        // Logic to update internal workflow state based on external events
    }
}
```

## Creating an Activity

Activities perform the actual "heavy lifting." They should be **idempotent**, as the engine may retry them on failure.

```php
namespace App\Account\Activities;

use Workflow\Contracts\Activity;
use Throwable;

class CreateUserRecord implements Activity
{
    // The engine automatically passes constructor data to the handle method
    public function __construct(protected array $payload) {}

    public function handle(array $payload): array
    {
        // ... Logic to create user in DB ...
        return ['id' => 123, 'status' => 'created'];
    }

    public function onFailure(string $instanceId, Throwable $e): void
    {
        // Cleanup logic if the activity fails after all retries
    }

    public function compensate(string $instanceId, array $originalPayload): void
    {
        // Saga pattern: Logic to "undo" this activity if the workflow fails later
    }
}
```

## Durable Helpers

The system provides several built-in helpers for common workflow needs.

### Timers & Delays

Workflow execution can be paused for minutes, days, or months.

```php
yield minutes(10);
yield days(3);
```

### Side Effects

For non-deterministic logic (like generating a random ID or getting the current time), use `sideEffect`. The result is recorded once and replayed thereafter.

```php
$token = yield sideEffect(fn() => bin2hex(random_bytes(16)));
```

### Inline Activities (Closures)

For trivial tasks, you can yield a closure directly.

```php
yield fn() => $this->logger->info("Processing step...");
```

## Activity Options

Configure retries, timeouts, and queues per-activity:

```php
use Workflow\Contracts\ActivityOptions;

yield new ProcessPayment($data, ActivityOptions::make()
    ->withTimeout(120)      // Max 120 seconds
    ->withRetries(5)        // Retry up to 5 times
    ->withRetryDelay(10)   // 10 seconds between retries
    ->onQueue('payments')   // Run on a specific queue
);
```

## Running a Workflow

```php
use Workflow\Workflow;

// 1. Run the workflow (creates instance and triggers first step)
$instanceId = Workflow::run(
    UserOnboardingWorkflow::class,
    ['email' => 'user@example.com'], // Input
    'user-123'                       // Business Key (Optional)
);

// 2. Resume or manually trigger execution (done automatically by the engine usually)
Workflow::execute($instanceId);
```

## How It Works: The Replay Magic

1. **Step Execution**: The engine runs the code until it hits a `yield`.
2. **Persistence**: The yielded Command is saved as an event. The workflow process **terminates**.
3. **External Action**: A worker (or timer) eventually completes the task.
4. **Resumption**: The engine re-instantiates the workflow.
5. **Deterministic Replay**: The engine runs the code **from line 1**. For every `yield` that was already completed, it simply returns the saved result instead of re-executing.
6. **Continuation**: When it reaches the new `yield`, it continues the cycle.

> **Constraint**: Because of Replay, workflow code MUST be deterministic. Do not use `rand()` or `time()` directly inside the `execute` method; use `sideEffect()` instead.

## Best Practices

- **Idempotent Activities**: Always assume your activity might run more than once.
- **Granular Activities**: Break large tasks into small, yieldable steps for better recovery.
- **Use Side Effects**: Wrap all non-deterministic functions in `sideEffect`.
- **Version Carefully**: If you change the code of a running workflow, the replay might fail (non-deterministic change). Use versioning flags for long-running workflows.
- **Business Keys**: Use meaningful business keys (like `order-456`) instead of random UUIDs for easier tracking.
