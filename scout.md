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
- Create necessary database tables (`scout_*`).
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

// Schedule an interview
Scout::scheduleInterview($application, $interviewer, [
    'scheduled_at' => '2026-02-01 10:00:00',
    'location' => 'Zoom'
]);

// Reject application
Scout::reject($application, 'Lack of required experience.');
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

## Integrations

- **Workflow**: Automated hiring pipeline transitions.
- **Slot**: Manage interviewer availability and bookings.
- **Link**: Signed, expiring links for offer letters.
- **Media**: Secure storage for resumes and portfolio documents.
- **Audit**: Comprehensive logs for all recruitment actions.
- **Onboard**: Seamless handoff when a candidate is hired.
