# Scheduling (Cron)

Anchor features a robust, class-based scheduling engine that abstracts away traditional procedural cron files. By leveraging a modular architecture, tasks are defined as first-class objects and discovered automatically across core application and package-specific schedules.

## Unified Background Processing

The framework uses a centralized `BackgroundDispatcher` to orchestrate all out-of-band tasks. This includes:

- **Queue Processing**: Re-queuing failed jobs and executing pending ones.
- **Task Scheduling**: Running periodic maintenance and business logic.
- **Deferred Tasks**: Executing post-response callback payloads.

This logic is triggered by a single entry point: `cron.php`.

## Getting Started

Scheduled tasks are classes that implement the `Cron\Interfaces\Schedulable` interface. They are automatically discovered by the framework based on their location.

### Discovery Locations

The framework scans the following directories for `Schedulable` classes:

- **Core App**: `App/Schedules/`
- **Internal Modules**: `App/src/{Module}/Schedules/`
- **Shared Packages**: `packages/{Package}/Schedules/`

## Creating a Schedule Class

You can use the `dock` CLI to generate a new schedule class:

```bash
# Create a global schedule
php dock schedule:create Backup

# Create a modular schedule in App/src/Account/Schedules/
php dock schedule:create Report Account
```

### Schedule Class Example

```php
namespace App\Schedules;

use Cron\Interfaces\Schedulable;
use Cron\Schedule;

class BackupSchedule implements Schedulable
{
    /**
     * Define the schedule for the task(s).
     */
    public function schedule(Schedule $schedule): void
    {
        // Run a CLI command daily at 2:00 AM
        $schedule->command('database:backup')->dailyAt('02:00');

        // Run a closure every hour
        $schedule->call(function() {
            // Your logic here...
        })->hourly();
    }
}
```

## Task Frequency Options

The `Schedule` object provides a comprehensive fluent API for defining task frequencies:

### Standard Intervals

- `->everyMinute()`: Every minute
- `->everyFiveMinutes()`: Every 5 minutes
- `->everyTenMinutes()`: Every 10 minutes
- `->everyFifteenMinutes()`: Every 15 minutes
- `->everyThirtyMinutes()`: Every 30 minutes
- `->hourly()`: Every hour
- `->hourlyAt(int $minute)`: Every hour at a specific minute (e.g., `->hourlyAt(15)`)
- `->daily()`: Every day at midnight
- `->dailyAt(string $time)`: Every day at a specific time (e.g., `->dailyAt('13:00')`)
- `->twiceDaily(int $first, int $second)`: Twice daily (default: 1 AM and 1 PM)

### Calendar & Days

- `->weekly()`: Every Sunday at midnight
- `->monthly()`: First day of every month at midnight
- `->monthlyOn(int $day, string $time)`: Specific day and time of the month
- `->weekdays()`: Monday through Friday
- `->weekends()`: Saturday and Sunday
- `->mondays()`, `->tuesdays()`, `->wednesdays()`, `->thursdays()`, `->fridays()`, `->saturdays()`, `->sundays()`

### Custom Range logic

- `->days(int|string|array $days)`: Specific days of the week (0-6)
- `->cron(string $expression)`: Raw cron expression support (e.g., `* * * * *`)

## Running the Dispatcher

The framework offers two ways to run your background tasks depending on your infrastructure needs.

### Unified Runner (Recommended)

Use `cron.php` for **Unified Background Processing**. This is the standard approach for most applications as it handles queues, schedules, and deferred tasks in a single entry point.

**Cron entry:**

```bash
* * * * * cd /path-to-your-project && php cron.php >> /dev/null 2>&1
```

**Use Case:** Small to medium applications that want a "set it and forget it" background system without managing multiple processes.

### Dedicated Scheduler

Use `php dock schedule:run` for **Dedicated Task Scheduling**. This only executes the `Cron` schedules and ignores the queue system.

**Cron entry:**

```bash
* * * * * cd /path-to-your-project && php dock schedule:run >> /dev/null 2>&1
```

**Use Case:** Large-scale applications where the queue system is handled by dedicated workers (e.g., `php dock queue:work`) and you want to isolate periodic maintenance tasks.

## Modular Package Scheduling

If you are developing a package, you can place your `Schedulable` classes in the `Schedules/` directory of your package (e.g., `packages/MyPackage/Schedules/`). The framework automatically discovers and registers them during the boot process.

### Naming Conventions

To ensure automatic discovery, your schedule classes must:

- Reside in a `Schedules` directory.
- Follow the `*Schedule.php` naming convention (e.g., `CleanupSchedule.php`).
- Implement the `Cron\Interfaces\Schedulable` interface.

> **Distinguishing from Queues**: While the `Queue` system handles immediate background jobs (e.g., sending an email after registration), the `Cron` system is intended for recurring maintenance (e.g., daily backups). `cron.php` handles both seamlessly.
