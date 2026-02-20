# Flow

**Flow** provides a highly customizable Project Management engine for the Anchor Framework. It allows you to build Kanban boards, list-based workflows, and complex task hierarchies with ease.

## Features

- **Dynamic Kanban Engine**: Fully configurable columns and workflow stages.
- **Hierarchical Tasks**: Support for parent-child task relationships (subtasks).
- **Collaboration First**: Built-in commenting, multiple assignees per task, and attachments.
- **Precision Scheduling**: Integrated recurrence rules for automated task creation.
- **Analytics & metrics**: Real-time project health reporting and burndown data.
- **Extensible Architecture**: Decoupled services for projects, tasks, and collaboration.

## Installation

Flow is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Flow --packages
```

This will automatically:

- Publish the configuration.
- Run the migration for Flow tables.
- Register the `FlowServiceProvider`.

### Configuration

Configuration is located at `App/Config/flow.php`. You can modify the default board structure and task priorities here.

| Key                    | Type     | Default        | Description                                     |
| :--------------------- | :------- | :------------- | :---------------------------------------------- |
| `flow.default_columns` | `array`  | `[...]`        | Columns created automatically for new projects. |
| `flow.priorities`      | `array`  | `['low', ...]` | Available priority levels for tasks.            |
| `flow.storage.disk`    | `string` | `'local'`      | The filesystem disk for attachments.            |
| `flow.storage.path`    | `string` | `'flow/...'`   | Path within the disk for storing files.         |

## Core Concepts

### Projects & Columns

A **Project** is the primary container. Each project owns multiple **Columns** (e.g., "Backlog", "In Review") which define the workflow. Tasks move between these columns to represent progress.

### Recursive Tasks

Tasks in Flow are recursive. A task can be a single unit of work or a container for multiple **Subtasks**. This allows for deep breakdown of complex requirements.

### Public Identifiers (refid)

All core Flow records (Projects, Tasks, Attachments, Comments) include a `refid`. This is a unique, non-enumeratable string (e.g., `8f7a2b...`) that is preferred over the numeric `id` for public-facing URLs and API lookups to enhance security.

## Basic Usage

### Initializing a Project

Use the `Flow` manager to create a new board. Default columns defined in your config will be added automatically.

```php
use Flow\Flow;

$project = Flow::projects()->make()
    ->name('Alpha Launch')
    ->description('Coordination for the Q4 product launch.')
    ->owner($user)
    ->save();
```

### Managing Tasks

Tasks are managed via the `TaskService`, accessible fluently through the `Flow` manager.

```php
use Flow\Flow;
use Flow\Models\Column;

// Create a task
$task = Flow::tasks()->make()
    ->title('Implement Auth Facade')
    ->project($project->id)
    ->priority('high')
    ->creator($user)
    ->save();

// Move to first position in target column
$targetColumn = Column::find($targetColumnId);
Flow::tasks()->move($task, $targetColumn, 0);

// Add dependency
Flow::tasks()->addDependency($task, $otherTask);

// Create subtask
$subtask = Flow::tasks()->make()
    ->title('Design Auth DB')
    ->project($project->id)
    ->parent($task->id)
    ->creator($user)
    ->save();
```

## Advanced Usage

### Collaboration & Assignees

Social interactions and task assignments are handled fluently through the `Flow` facade.

```php
use Flow\Flow;

// Assign users
Flow::tasks()->addAssignee($task, $devUser);

// Add context via comments
Flow::collaboration()->addComment($task, $user, 'Blocked by API availability.');

// Attach design files fluently
Flow::collaboration()->makeAttachment()
    ->to($task)
    ->by($user)
    ->path('uploads/design.png')
    ->filename('design.png')
    ->mime('image/png')
    ->size(1024)
    ->save();
```

### Recurring Workflows & Reporting

Automate repetitive tasks and monitor project health through the reporting facade.

```php
use Flow\Flow;

// Setup recurring task fluently
Flow::recurring()->for($task)
    ->weekly() // Options: daily(), weekly(), monthly()
    ->startingAt('next Monday') // Accepts any date string or relative string (e.g., '2025-01-01')
    ->save();

// Get analytics fluently
$completionRate = Flow::reports()->for($project)->completionRate();
$burndown = Flow::reports()->for($project)->burndown();
```

#### Expected Return Values

- **`completionRate()`**: Returns a `float` (e.g., `75.5`) representing the percentage of tasks in "done" columns.
- **`burndown()`**: Returns an `array` mapping dates to remaining tasks for the last 30 days.
  ```php
  [
      "2023-10-01" => 25,
      "2023-10-02" => 22,
      // ...
  ]
  ```
- **`kanbanData()`**: Returns the board structure with columns and their tasks.
  ```php
  [
      [
          'id' => 1,
          'name' => 'Backlog',
          'type' => 'backlog',
          'tasks' => [
              ['id' => 1, 'title' => 'Design UI', 'priority' => 'high', ...]
          ]
      ],
      [
          'id' => 2,
          'name' => 'Done',
          'type' => 'done',
          'tasks' => []
      ]
  ]
  ```
- **`userTaskStats()`**: Returns per-user task statistics.
  ```php
  [
      [
          'user' => ['id' => 1, 'name' => 'Alice', ...],
          'total' => 5,
          'completed' => 3,
          'overdue' => 1
      ],
      [
          'user' => ['id' => 2, 'name' => 'Bob', ...],
          'total' => 2,
          'completed' => 2,
          'overdue' => 0
      ]
  ]
  ```
- **`taskDistribution('status')`**: Returns task counts grouped by column name.
  ```php
  [
      'Backlog' => 5,
      'In Progress' => 3,
      'Done' => 10
  ]
  ```

#### Chart-Ready Data

#### Library Agnostic

The following methods return a universal data structure compatible with any charting library (Chart.js, ApexCharts, Highcharts, D3, etc.):

- **`burndownChart()`**: Line chart data for project burndown.
  ```php
  [
      'labels' => ['2023-10-01', '2023-10-02', ...],
      'datasets' => [
          ['label' => 'Remaining Tasks', 'data' => [25, 22, 18, ...]]
      ]
  ]
  ```
- **`distributionChart('status')`**: Pie/bar chart data for task distribution.
  ```php
  [
      'labels' => ['Backlog', 'In Progress', 'Done'],
      'datasets' => [
          ['label' => 'Tasks by Status', 'data' => [5, 3, 10]]
      ]
  ]
  ```
- **`userStatsChart()`**: Grouped bar chart for comparing user performance.
  ```php
  [
      'labels' => ['Alice', 'Bob'],
      'datasets' => [
          ['label' => 'Total', 'data' => [5, 2]],
          ['label' => 'Completed', 'data' => [3, 2]],
          ['label' => 'Overdue', 'data' => [1, 0]]
      ]
  ]
  ```

### Reminders & Notifications

Set up reminders for tasks and receive email notifications when they're due.

```php
use Flow\Flow;

// Create a reminder
Flow::reminders()->make()
    ->for($task)
    ->user($user)         // Also supports notify($user)
    ->at('next Friday')   // Supports relative strings or absolute dates
    ->save();
```

Flow dispatches the following notifications:

- **`TaskAssignedNotification`** - Sent when a user is assigned to a task
- **`TaskReminderNotification`** - Sent when a reminder triggers

### CLI Commands

Flow provides commands to process asynchronous tasks:

```bash
# Process all due reminders
php dock flow:remind

# Process and spawn recurring tasks
php dock flow:recur
```

### Fetching Records by refid

While `id` is used for internal relationships, always use `refid` for public-facing lookups.

```php
use Flow\Flow;

// Find a project
$project = Flow::projects()->findByRefid('proj_abc123');

// Find a task
$task = Flow::tasks()->findByRefid('task_xyz789');

// Find collaboration assets
$comment = Flow::collaboration()->findCommentByRefid('comm_123');
$attachment = Flow::collaboration()->findAttachmentByRefid('att_456');
```

### Tags & Metadata

Organize tasks across projects using the global tagging system fluently.

```php
use Flow\Flow;

// Attach tag by name (creates if doesn't exist)
Flow::tasks()->addTag($task, 'Urgent');
```

## Use Case

#### Case Study: Marketing Campaign

This example demonstrates how the Flow API comes together to manage a high-stakes campaign launch.

```php
use Flow\Flow;

// 1. Initialize the Campaign Project
$campaign = Flow::projects()->make()
    ->name('Q1 Product Launch')
    ->description('Coordination of social media, email, and ad spend.')
    ->owner($manager)
    ->save();

// 2. Create the "Hero" Task
$mainTask = Flow::tasks()->make()
    ->title('Social Media Teaser Video')
    ->project($campaign->id)
    ->priority('urgent')
    ->creator($manager)
    ->save();

// 3. Add a Recursive Subtask (Weekly check-in)
$checkIn = Flow::tasks()->make()
    ->title('Weekly Analytics Review')
    ->project($campaign->id)
    ->parent($mainTask->id)
    ->creator($manager)
    ->save();

Flow::recurring()->for($checkIn)
    ->weekly()
    ->startingAt('next Friday')
    ->save();

// 4. Collaboration: Assign a Designer & Attach Assets
Flow::tasks()->addAssignee($mainTask, $designer);
Flow::tasks()->addTag($mainTask, 'Creative');

Flow::collaboration()->makeAttachment()
    ->to($mainTask)
    ->by($designer)
    ->path('branding/teaser_v1.mp4')
    ->filename('teaser_v1.mp4')
    ->mime('video/mp4')
    ->save();

// 5. Track Health
$health = Flow::reports()->for($campaign)->completionRate();
```

## Service API Reference

### ProjectService

| Method                                               | Description                                |
| :--------------------------------------------------- | :----------------------------------------- |
| `make()`                                             | Returns a `ProjectBuilder` instance.       |
| `create(array $data, User $owner)`                   | Creates a project and its default columns. |
| `update(Project $project, array $data)`              | Updates project attributes.                |
| `findByRefid(string $refid)`                         | Finds a project by its public identifier.  |
| `delete(Project $project)`                           | Deletes a project.                         |
| `createColumn(Project $project, array $data)`        | Adds a custom column to a project.         |
| `updateColumnOrder(Project $project, array $orders)` | Bulk updates column positions.             |

### TaskService

| Method                                      | Description                                             |
| :------------------------------------------ | :------------------------------------------------------ |
| `make()`                                    | Returns a `TaskBuilder` instance.                       |
| `create(array $data, User $creator)`        | Creates a task (or subtask if `parent_id` is provided). |
| `update(Task $task, array $data)`           | Updates task attributes.                                |
| `findByRefid(string $refid)`                | Finds a task by its public identifier.                  |
| `move(Task $task, Column $col, int $order)` | Moves a task and handles automatic re-ordering.         |
| `addAssignee/removeAssignee`                | Manage users assigned to the task.                      |
| `addDependency/removeDependency`            | Manage task blocking relationships.                     |
| `addTag/removeTag`                          | Manage task labels (supports string or Tag object).     |

### CollaborationService

| Method                                      | Description                                   |
| :------------------------------------------ | :-------------------------------------------- |
| `makeAttachment()`                          | Returns an `AttachmentBuilder` instance.      |
| `addComment(Task $t, User $u, string $c)`   | Adds a comment to a task.                     |
| `findCommentByRefid(string $refid)`         | Finds a comment by its public identifier.     |
| `attachFile(Task $t, User $u, array $data)` | Uploads and associates a file with a task.    |
| `findAttachmentByRefid(string $refid)`      | Finds an attachment by its public identifier. |
| `removeAttachment(Attachment $a)`           | Deletes the record and the physical file.     |

### Reporting Service

| Method                                            | Description                                               |
| :------------------------------------------------ | :-------------------------------------------------------- |
| `for(Project $p)`                                 | Returns a `ReportingProxy` for fluent metrics.            |
| `getProjectCompletionRate(Project $p)`            | Returns percentage of tasks in 'done' columns.            |
| `getBurndownData(Project $p)`                     | Returns daily remaining task counts for the last 30 days. |
| `getKanbanData(Project $p)`                       | Returns board structure with columns and their tasks.     |
| `getUserTaskStats(Project $p)`                    | Returns per-user task counts (total, done, overdue).      |
| `getTaskDistribution(Project $p, string $g)`      | Returns task counts grouped by column type.               |
| `getBurndownChartData(Project $p)`                | Returns burndown data formatted for charts.               |
| `getDistributionChartData(Project $p, string $g)` | Returns distribution data formatted for charts.           |
| `getUserStatsChartData(Project $p)`               | Returns user stats formatted for charts.                  |

### RecurringTaskService

| Method                           | Description                                         |
| :------------------------------- | :-------------------------------------------------- |
| `for(Task $t)`                   | Returns a `RecurringBuilder` instance.              |
| `processRecurringTasks()`        | Background worker to spawn new instances.           |
| `createNextInstance(Task $task)` | Manually trigger the creation of the next instance. |

### ReminderService

| Method                | Description                            |
| :-------------------- | :------------------------------------- |
| `make()`              | Returns a `ReminderBuilder` instance.  |
| `create(array $data)` | Creates a reminder directly from data. |
| `processReminders()`  | Processes and sends all due reminders. |
 
## Automation
 
Flow uses automated scheduling to process task recurrence and dispatch reminders. These tasks are automatically registered in the central scheduler:
 
```php
// packages/Flow/Schedules/FlowSchedule.php
namespace Flow\Schedules;
 
use Cron\Interfaces\Schedulable;
use Cron\Schedule;
 
class FlowSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('flow:recur')
            ->hourly();
 
        $schedule->task()
            ->signature('flow:reminders')
            ->everyMinute();
    }
}
```

## Best Practices

- Always customize `flow.default_columns` to match your organization's specific workflow before bulk-creating projects. This ensures consistency across your workspace.

- Use the `TaskService::move()` method instead of manual model updates. This method ensures that column orders are recalculated and all necessary events are triggered.

- While tasks are recursive, it is recommended to keep subtask depth to 2-3 levels for better UI performance and readability.
