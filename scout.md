# Scout

Scout is a comprehensive Applicant Tracking System (ATS) for the Anchor Framework. It provides tools for managing job postings, candidate pipelines, interview scheduling, and recruitment analytics.

## Features

- **Job Management**: Create, publish, and track various job types (full-time, remote, etc.).
- **Candidate Pipeline**: Manage candidates through customizable recruitment stages.
- **Interview Scheduling**: Coordinate with the `Slot` package to book interviewers.
- **Offer Engine**: Generate secure offer links via the `Link` package.
- **Unified Analytics**: Track funnel conversion rates and time-to-hire metrics.
- **Recruiter Collaboration**: Private notes and scorecard reviews for hiring teams.

## Installation

Scout is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Scout --packages
```

This command will:

- Publish the `scout.php` configuration file.
- Run the migration for Scout tables.
- Register the `ScoutServiceProvider`.

## Facade API

### Job Management

```php
use Scout\Scout;

// Create and publish a job
$job = Scout::job()
    ->title('Senior Backend Engineer')
    ->description('Lead our core API development.')
    ->location('Remote')
    ->publish()
    ->create();
```

### Candidate & Application

```php
// Create a candidate
$candidate = Scout::candidate()
    ->name('John Doe')
    ->email('john@example.com')
    ->talentPool()
    ->create();

// Apply for a job
$application = Scout::application()
    ->for($job)
    ->from($candidate)
    ->submit();
```

### Pipeline Orchestration

```php
// Advance to next stage
Scout::advance($application, $nextStage);

// Schedule an interview (optionally linked to a Slot)
Scout::scheduleInterview($application, $interviewer, [
    'scheduled_at' => '2026-02-01 10:00:00',
    'location' => 'Zoom',
    'slot_id' => 123, // Link to a Slot booking for availability tracking
]);

// Retrieve the booking details using the Slot facade (decoupled)
use Slot\Slot;

if ($interview->slot_id) {
    $booking = Slot::getBooking($interview->slot_id);
}

// Reject application
Scout::reject($application, 'Lack of required experience.');
```

### Offer Management

```php
// Create and send an offer
$offer = Scout::createOffer($application, 85000, [
    'currency' => 'USD',
    'terms' => 'Stock options included',
    'expires_at' => '2026-02-15 17:00:00',
]);

// The offer link is automatically generated if the Link package is installed.
```

### Stage Management

You can manage recruitment stages using the `Scout` facade.

```php
// List all stages
$stages = Scout::stages();

// Fetch a specific stage by slug or ID
$stage = Scout::findStage('technical-interview');

// Create a new stage
$stage = Scout::createStage([
    'name' => 'Technical Challenge',
    'order' => 2,
]);

// Update a stage
Scout::updateStage($stage, [
    'name' => 'Technical Interview - Level 1',
]);

// Delete a stage
Scout::deleteStage($stage);
```

### Analytics

```php
$analytics = Scout::analytics();

// Returns: [['name' => 'Applied', 'count' => 50], ['name' => 'Interview', 'count' => 10], ...]
$funnel = $analytics->funnel($job);

// Returns: ['total_jobs' => 10, 'active_jobs' => 5, 'total_applicants' => 100, ...]
$overview = $analytics->overview();

// Returns: (float) 14.5
$avgTime = $analytics->averageTimeToHire();
```

## Interview Reminders

Scout includes a built-in reminder system for upcoming interviews.

### Command Line

You can send reminders for interviews starting within the next 30 minutes using the CLI:

```bash
php dock scout:reminders
```

### Programmatic Reminders

You can also trigger reminders via the facade:

```php
use Scout\Scout;

$count = Scout::sendReminders();
echo "Sent {$count} reminders.";
```

### Automation

Scout automatically registers its reminder command in the global scheduler. The `ScoutSchedule` class runs the `scout:reminders` command every thirty minutes to ensure candidates and interviewers receive timely notifications.

```php
// packages/Scout/Schedules/ScoutSchedule.php
namespace Scout\Schedules;

use Cron\Interfaces\Schedulable;
use Cron\Schedule;

class ScoutSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->command('scout:reminders')->everyThirtyMinutes();
    }
}
```

## Integrations

- **Workflow**: Automated hiring pipeline transitions.
- **Slot**: Manage interviewer availability and bookings.
- **Link**: Signed, expiring links for offer letters.
- **Media**: Secure storage for resumes and portfolio documents.
- **Audit**: Comprehensive logs for all recruitment actions.
- **Onboard**: Seamless handoff when a candidate is hired.
