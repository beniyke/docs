# Metric

Metric is the performance management engine for the Anchor Framework. It enables organizations to track goals (OKRs), monitor KPIs, orchestrate review cycles, and foster a culture of recognition.

## Features

- **Goal Management (OKRs)**: Define objectives and quantifiable key results with automated progress tracking.
- **KPI Engine**: Track periodic metrics globally or per user with custom measurement units.
- **Review Cycles**: Manage annual, quarterly, or custom performance review workflows.
- **360 Feedback**: Collect peer, manager, and self-assessments (with anonymity support).
- **1-on-1 Scheduler**: Coordinate regular check-ins with integrated scheduling (via `Slot`).
- **Recognition & Kudos**: Encourage peer-to-peer appreciation with trackable awards.
- **Performance Analytics**: Benchmark teams and identify top performers via aggregate data.

## Installation

Metric is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Metric --packages
```

This command will:

- Publish the `metric.php` configuration file.
- Create necessary database tables (`metric_*`).
- Register the `MetricServiceProvider`.

## Facade API

### OKR Tracking

```php
use Metric\Metric;

// Create a goal with key results
$goal = Metric::goal()
    ->for($user)
    ->title('Scale Infrastructure')
    ->addKeyResult('Achieve 99.9% uptime', 99.9)
    ->addKeyResult('Reduce latency by 20%', 20.0, 'percentage')
    ->create();

// Update progress
Metric::updateKeyResult($keyResult, 95.0);
```

### KPI Management

```php
// Define and record a KPI
$kpi = Metric::kpi()->name('Sales Volume')->unit('USD')->create();
Metric::recordKpi($kpi, 15000.00, $salesRep);
```

### Feedback & Recognition

```php
// Give kudos
Metric::recognize($sender, $receiver, 'excellence', 'Fantastic job on the Q1 release!');

// Submit feedback
Metric::giveFeedback($manager, $employee, 'Consistently exceeds expectations in code quality.');
```

### Analytics

```php
$stats = Metric::analytics()->goalOverview();
// Returns: ['total_goals' => 50, 'average_progress' => 65.5, 'completed_goals' => 10, 'overdue_goals' => 2]

$topPerformers = Metric::analytics()->topPerformers(10);
// Returns: [['user_id' => 1, 'avg_rating' => 4.8, 'user' => [...]], ...]
```

## Integrations

- **Onboard**: Initialize default performance goals for new hires.
- **Flow**: (Planned) Link specific project tasks to performance objectives.
- **Slot**: Book 1-on-1 meetings using the central scheduling system.
- **Hub**: Create announcement threads for team recognitions.
- **Audit**: Immutable history of performance appraisals and goal adjustments.
- **Workflow**: Orchestrate review cycle transitions (Draft -> Active -> Finalized).
