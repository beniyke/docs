# Queues

Anchor provides a robust queue system for processing background tasks with integrated scheduling capabilities. The scheduling system allows you to delay job execution using a fluent API for calculating future dates and times.

## Installation

The Queue system is a core package that must be installed before use:

```bash
php dock package:install queue --system
```

This command will:

- Publish the `queue.php` configuration file.
- Publish and run the database migrations for the `queued_job` table.
- Register the `QueueServiceProvider` in your application.

## Core Concepts

- **Queue Manager**: The entry point for dispatching jobs (`Queue\QueueManager`).
- **Queue Dispatcher**: The worker that processes jobs (`Queue\QueueDispatcher`).
- **Scheduler**: A helper for calculating future execution times (`Queue\Scheduler`).
- **QueuedJob**: The database model representing a job (`Queue\Models\QueuedJob`).

## Creating a Job

A job is a simple class that implements the `Queue\Interfaces\Queueable` interface. It must define an `execute()` method (inherited from `BaseTask` or implemented directly).

### Task Locations

Anchor supports two patterns for organizing your tasks:

- **Modular Tasks (Recommended)**: Tasks specific to a feature (e.g., `Auth`, `Billing`) should live in your module's source directory:
  - **Path**: `App/src/{Module}/Tasks/MyTask.php`
  - **Namespace**: `App\{Module}\Tasks`
- **Global/Shared Tasks**: Cross-cutting tasks (e.g., `DailyBackup`, `SystemCleanup`) can live in a global directory:
  - **Path**: `App/Tasks/MyTask.php`
  - **Namespace**: `App\Tasks`

> The `task:create` CLI command defaults to the **Modular** pattern and automatically appends the `Task` suffix to your class name.

### Using the CLI

You can generate and manage your task classes using the following commands:

```bash
# Create modular task "SendWelcomeEmailTask" in "Auth" module
php dock task:create SendWelcomeEmail Auth

# Create global task "DailyBackupTask" in "App/Tasks"
php dock task:create DailyBackup
```

> You do not need to include "Task" in the name; the framework appends it automatically.

To delete a task:

```bash
# Delete modular task
php dock task:delete SendWelcomeEmail Auth

# Delete global task
php dock task:delete DailyBackup
```

**Example Task Class**

```php
namespace App\Auth\Tasks;

use Queue\BaseTask;
use Queue\Scheduler;

class SendWelcomeEmailTask extends BaseTask
{
    public function occurrence(): string
    {
        return self::once();
    }

    public function period(Scheduler $schedule): Scheduler
    {
        // Run immediately (default)
        return $schedule;
    }

    protected function execute(): bool
    {
        $userId = $this->payload['user_id'];
        // Logic to send email...

        return true; // Return true on success
    }

    protected function successMessage(): string
    {
        return 'Welcome email sent successfully';
    }

    protected function failedMessage(): string
    {
        return 'Failed to send welcome email';
    }
}
```

## Dispatching Jobs

You can dispatch jobs using the global `queue()` helper or the `QueueManager` directly.

### Immediate Execution (Async)

To queue a job to run as soon as a worker picks it up:

```php
use App\Auth\Tasks\SendWelcomeEmailTask;
use Queue\QueueManager;

// Using the helper
queue(SendWelcomeEmailTask::class, ['user_id' => 1]);

// Using the manager
resolve(QueueManager::class)
    ->job(SendWelcomeEmailTask::class, ['user_id' => 1])
    ->queue();
```

### Custom Queues

You can organize jobs into different queues (identifiers) to prioritize processing.

```php
use App\Auth\Tasks\CriticalOperationTask;
use Queue\QueueManager;

// Using the helper with identifier
queue(CriticalOperationTask::class, $data, 'high-priority');

// Using the manager
resolve(QueueManager::class)
    ->identifier('high-priority')
    ->job(CriticalOperationTask::class, $data)
    ->queue();
```

### Using the Facade (Recommended)

The `Queue` facade provides a clean, static interface for dispatching jobs.

```php
use App\Auth\Tasks\SendWelcomeEmailTask;
use App\Auth\Tasks\CriticalOperationTask;
use Queue\Queue;

// Simple dispatch
Queue::dispatch(SendWelcomeEmailTask::class, ['user_id' => 1]);

// Dispatch to a specific queue
Queue::dispatch(CriticalOperationTask::class, $data, 'high-priority');
```

## Scheduling Jobs

The `Queue\Scheduler` class provides a fluent interface for calculating when tasks should run in the future. You can schedule jobs by implementing the `period()` method in your job class.

### Delayed Execution with period()

Implement the `period()` method in your job class to define when the job should run:

```php
namespace App\Tasks;

use Queue\BaseTask;
use Queue\Scheduler;

class ProcessSubscriptionRenewalTask extends BaseTask
{
    public function occurrence(): string
    {
        return self::once();
    }

    public function period(Scheduler $schedule): Scheduler
    {
        // Run this job 30 days from now
        return $schedule->day(30);
    }

    protected function execute(): bool
    {
        // Process renewal logic...
        return true;
    }

    protected function successMessage(): string
    {
        return 'Subscription renewal processed';
    }

    protected function failedMessage(): string
    {
        return 'Failed to process subscription renewal';
    }
}
```

When you dispatch this job, the framework automatically calculates the `schedule` timestamp based on your `period()` method:

```php
queue(ProcessSubscriptionRenewalTask::class, ['subscription_id' => 123]);
```

## Scheduling API Reference

### Time Period Methods

The Scheduler provides methods for adding various time periods to calculate future execution times.

#### Minutes

```php
$scheduler = resolve(Scheduler::class);

// 30 minutes from now
$time = $scheduler->minute(30)->time();

// 1 minute from now (default)
$time = $scheduler->minute()->time();
```

#### Days

```php
// 7 days from now
$time = $scheduler->day(7)->time();

// 1 day from now (default)
$time = $scheduler->day()->time();
```

#### Weeks

```php
// 2 weeks from now
$time = $scheduler->week(2)->time();

// 1 week from now (default)
$time = $scheduler->week()->time();
```

#### Months

```php
// 3 months from now
$time = $scheduler->month(3)->time();

// 1 month from now (default)
$time = $scheduler->month()->time();
```

#### Years

```php
// 2 years from now
$time = $scheduler->year(2)->time();

// 1 year from now (default)
$time = $scheduler->year()->time();
```

### Chaining Time Periods

You can chain multiple time periods together for complex scheduling:

```php
$scheduler = resolve(Scheduler::class);

// 1 month, 2 weeks, and 3 days from now
$time = $scheduler
    ->month(1)
    ->week(2)
    ->day(3)
    ->time();

// 1 year, 6 months, and 15 days from now
$time = $scheduler
    ->year(1)
    ->month(6)
    ->day(15)
    ->time();
```

### Setting Initial Date

By default, the scheduler starts from the current time. You can set a custom initial date:

```php
use Helpers\DateTimeHelper;

$scheduler = resolve(Scheduler::class);

// Start from a specific date
$initialDate = DateTimeHelper::parse('2024-01-01 00:00:00');
$scheduler->setInitialDate($initialDate);

// Calculate 30 days from that date
$futureDate = $scheduler->day(30)->time();
```

### Scheduling at Specific Times

The `at()` method allows you to schedule tasks at specific times of day.

#### Basic Usage

```php
$scheduler = resolve(Scheduler::class);

// Tomorrow at 9:00 AM
$time = $scheduler->day(1)->at('09:00')->time();

// Next week at 2:30 PM
$time = $scheduler->week(1)->at('14:30')->time();

// With seconds
$time = $scheduler->day(1)->at('14:30:45')->time();
```

#### Time Format

The `at()` method accepts time in `HH:MM` or `HH:MM:SS` format (24-hour):

```php
// Valid formats
$scheduler->at('09:00');      // 9:00 AM
$scheduler->at('14:30');      // 2:30 PM
$scheduler->at('23:59:59');   // 11:59:59 PM
$scheduler->at('00:00');      // Midnight
```

#### Validation

The `at()` method validates time components and throws exceptions for invalid input:

```php
// These will throw InvalidArgumentException
$scheduler->at('25:00');      // Invalid hour (must be 0-23)
$scheduler->at('12:60');      // Invalid minute (must be 0-59)
$scheduler->at('12:30:60');   // Invalid second (must be 0-59)
$scheduler->at('invalid');    // Invalid format
```

### Timezone Support

The scheduler respects your application's configured timezone:

```php
// Timezone is set from config automatically
$scheduler = resolve(Scheduler::class);

// All calculations use the configured timezone
$time = $scheduler->day(1)->time();
```

### Using with DateTimeHelper

The scheduler returns `DateTimeHelper` objects, which provide additional functionality:

```php
$scheduler = resolve(Scheduler::class);
$futureDate = $scheduler->day(7)->time();

// Format the date
$formatted = $futureDate->format('Y-m-d H:i:s');

// Get timestamp
$timestamp = $futureDate->timestamp();

// Compare dates
if ($futureDate->isAfter($someOtherDate)) {
    // Future date is after the other date
}

// Convert to different timezone
$futureDate->setTimezone('America/New_York');
```

## Recurring Tasks

Anchor handles recurring tasks by having a job **reschedule itself** upon completion. This is done by returning `self::always()` from the `occurrence()` method. You can also use `self::once()` for non-recurring tasks (default).

```php
namespace App\Tasks;

use Queue\BaseTask;
use Queue\Scheduler;

class DailyReportTask extends BaseTask
{
    public function occurrence(): string
    {
        return self::always();
    }

    public function period(Scheduler $schedule): Scheduler
    {
        // Run every 24 hours
        return $schedule->day(1);
    }

    protected function execute(): bool
    {
        // Generate report...
        return true;
    }

    protected function successMessage(): string
    {
        return 'Daily report generated';
    }

    protected function failedMessage(): string
    {
        return 'Failed to generate daily report';
    }
}
```

To start the cycle, you only need to dispatch the job **once**. After it executes successfully, the `QueueDispatcher` will automatically create a new job record scheduled for the next interval.

## Scheduling Examples

### Daily Report at 9 AM

```php
namespace App\Tasks;

use Queue\BaseTask;
use Queue\Scheduler;

class SendDailyReportTask extends BaseTask
{
    public function occurrence(): string
    {
        return self::always();
    }

    public function period(Scheduler $schedule): Scheduler
    {
        return $schedule->day(1)->at('09:00');
    }

    protected function execute(): bool
    {
        // Generate and send report
        return true;
    }

    protected function successMessage(): string
    {
        return 'Daily report sent successfully';
    }

    protected function failedMessage(): string
    {
        return 'Failed to send daily report';
    }
}

// Dispatch once to start the recurring cycle
queue(SendDailyReportTask::class, []);
```

### Weekly Backup at 2 AM

```php
namespace App\Tasks;

use Queue\BaseTask;
use Queue\Scheduler;

class WeeklyBackupTask extends BaseTask
{
    public function occurrence(): string
    {
        return self::always();
    }

    public function period(Scheduler $schedule): Scheduler
    {
        return $schedule->week(1)->at('02:00');
    }

    protected function execute(): bool
    {
        // Perform backup
        return true;
    }

    protected function successMessage(): string
    {
        return 'Weekly backup completed';
    }

    protected function failedMessage(): string
    {
        return 'Weekly backup failed';
    }
}
```

### Monthly Invoice on 1st at 8 AM

```php
use App\Tasks\GenerateMonthlyInvoicesTask;
use Helpers\DateTimeHelper;
use Queue\Scheduler;

$scheduler = resolve(Scheduler::class);

// Set to first of next month
$firstOfMonth = DateTimeHelper::parse('first day of next month');
$scheduler->setInitialDate($firstOfMonth);

$invoiceTime = $scheduler->at('08:00')->time();

queue(GenerateMonthlyInvoicesTask::class, [], 'default');
```

### Event Reminder 1 Hour Before

```php
use App\Tasks\SendEventReminderTask;
use Helpers\DateTimeHelper;
use Queue\Scheduler;

$eventTime = DateTimeHelper::parse($event->start_time);

// 1 hour before the event
$scheduler = resolve(Scheduler::class);
$scheduler->setInitialDate($eventTime);
$reminderTime = $scheduler->minute(-60)->time();

queue(SendEventReminderTask::class, [
    'event_id' => $event->id
]);
```

### Subscription Renewal with Reminder

```php
use App\Tasks\SubscriptionRenewalReminderTask;
use Queue\Scheduler;

$scheduler = resolve(Scheduler::class);

// 1 month from now
$renewalDate = $scheduler->month(1)->time();

// Queue renewal reminder 3 days before
$reminderDate = $scheduler
    ->month(1)
    ->day(-3) // Subtract 3 days
    ->time();

queue(SubscriptionRenewalReminderTask::class, [
    'user_id' => $userId,
    'renewal_date' => $renewalDate->format('Y-m-d'),
]);
```

### Calculate Trial Expiration

```php
use Queue\Scheduler;

$scheduler = resolve(Scheduler::class);

// 14-day trial
$trialExpires = $scheduler->day(14)->time();

// Save to database
$user->trial_expires_at = $trialExpires->format('Y-m-d H:i:s');
$user->save();
```

## Processing & Managing the Queue

To process jobs, you need to run the queue worker command. This is typically run as a daemon process (e.g., using Supervisor).

### Running the Worker

Process queued jobs using `queue:run`.

```bash
# Process all pending jobs and retry failed ones (default)
php dock queue:run

# Process only pending jobs
php dock queue:run pending

# Retry only failed jobs
php dock queue:run failed

# Process jobs for a specific queue identifier
php dock queue:run --identifier=high-priority
```

### Clearing the Queue

Remove jobs from the queue using `queue:flush`.

```bash
# Flush all jobs (use with caution!)
php dock queue:flush

# Flush only failed jobs
php dock queue:flush failed

# Flush only pending jobs
php dock queue:flush pending

# Flush jobs for a specific identifier
php dock queue:flush --identifier=high-priority
```

## Configuration

Queue settings are defined in your configuration files (accessed via `ConfigService`).

- `queue.batch_size`: Number of jobs to retrieve at once (default: 10).
- `queue.max_retry`: Maximum number of retry attempts for failed jobs (default: 3).
- `queue.timeout_minutes`: Time before a reserved job is considered "stuck" and released (default: 5).
- `queue.delay_minutes`: Delay before retrying a failed job (default: 5).
- `queue.check_state`: Enable/disable queue pause check to improve performance if global pauses are not needed (default: true).

## Best Practices

1. **Use appropriate time units**: Choose the most readable unit (days vs hours vs minutes).
2. **Consider timezones**: Be aware of timezone differences for global applications.
3. **Validate calculations**: Ensure calculated dates make sense for your use case.
4. **Chain thoughtfully**: Keep chains readable and logical.
5. **Document scheduled tasks**: Comment why tasks are scheduled at specific intervals.
6. **Handle failures gracefully**: Implement proper error handling in your `execute()` method.
7. **Use occurrence wisely**: Only use `self::always()` for truly recurring tasks. Use `self::once()` otherwise.
8. **Monitor queue workers**: Ensure your queue workers are running in production.

## Deferred Dispatching

For improved performance, you can use `Queue::deferred()` to schedule a job to be dispatched _after_ the HTTP response has been sent to the user. This is a "fire-and-forget" approach that speeds up the user experience by moving the database insertion of the job to the end of the request lifecycle.

```php
use Queue\Queue;
use App\Jobs\ProcessReport;

// Job will be dispatched to the database after the response is sent
Queue::deferred(ProcessReport::class, ['report_id' => 123]);
```

> Since the job dispatching is deferred, `Queue::deferred()` returns `void` and does not provide the created `QueuedJob` instance or ID immediately.
