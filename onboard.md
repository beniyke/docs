# Onboard

Onboard is a core people-management package for the Anchor Framework. It automates employee onboarding through roles-based templates, document verification, training modules, and equipment tracking.

## Features

- **Template Engine**: Create reusable onboarding workflows for different roles (e.g., Engineering, Sales).
- **Task Management**: Structured checklists with required and optional tasks.
- **Document Collection**: Securely collect and verify identity documents and signed contracts.
- **Training Tracking**: Monitor progress through mandatory and elective training modules.
- **Equipment Provisioning**: Track asset requests like laptops and access badges.
- **Analytics**: Real-time progress tracking and completion benchmarks.

## Installation

Onboard is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Onboard --packages
```

This command will:

- Publish the `onboard.php` configuration file.
- Run the migration for Onboard tables.
- Register the `OnboardServiceProvider`.

### Configuration

Configuration file: `App/Config/onboard.php`

Define your preferences in the config file to customize:

- Default due dates.
- Manager notification settings.
- Integration triggers.

## Facade API

### Starting Onboarding

```php
use Onboard\Onboard;

// Start onboarding for a new hire
$onboarding = Onboard::onboarding()
    ->for($user)
    ->using($engineeringTemplate)
    ->dueAt($oneMonthFromNow)
    ->start();
```

### Task Management

Onboard provides a fluent `TaskBuilder` to create tasks dynamically or within templates.

```php
// Create a new task fluently
$task = Onboard::task()
    ->name('Setup Workspace')
    ->description('Complete IT setup and workspace preparation.')
    ->template($engineeringTemplate)
    ->order(1)
    ->create();

// Mark task as complete
Onboard::completeTask($user, $task, 'Workspace is ready.');
```

### Document Management

```php
// Track required documents
$document = $template->documents()->create([
    'name' => 'Passport',
    'is_required' => true
]);

// Verify a document upload
Onboard::verifyDocument($documentUpload, $hrUser);
```

### Training

```php
// Update training progress
use Onboard\Enums\TrainingStatus;
Onboard::training()->updateProgress($user, $safetyTraining, TrainingStatus::IN_PROGRESS); // or TrainingStatus::COMPLETED
```

### Equipment Provisioning

Track asset requests like laptops, phones, and access badges.

```php
use Onboard\Onboard;
 
// Request equipment
$equipment = Onboard::equipment()->request($user, 'Laptop', 'MacBook Pro 14"');
 
// Update status / Assign asset tag
use Onboard\Enums\EquipmentStatus;
Onboard::equipment()->assign($equipment, 'ASSET-123', EquipmentStatus::DELIVERED);
```

## Automatic Completion

The system automatically monitors progress through internal service triggers. When all **required tasks** are completed and **required documents** are verified by a reviewer, the `OnboardManagerService` automatically updates the onboarding status to `OnboardStatus::COMPLETED` and initiates a handoff to the `Metric` package.

## Analytics

Track onboarding health and individual progress.

```php
$stats = Onboard::analytics()->overview();
// Returns: ['total_onboarding' => 10, 'completed' => 5, 'in_progress' => 3, 'overdue' => 2]

$userProgress = Onboard::analytics()->progress($userId);
// Returns: (float) 75.5
```

## Integrations

- **Scout**: (Planned) Automatically initiate onboarding when a candidate is hired.
- **Flow**: (Planned) Sync onboarding tasks with the global task management system.
- **Media**: Secure storage for all uploaded documents.
- **Audit**: Immutable trail of every completion and verification event.
- **Metric**: (Planned) Automatically trigger goal initialization when onboarding is 100% complete.

## Models

Onboard relies on several core models to manage the workflow:

| Model              | Description                                                                 |
| :----------------- | :-------------------------------------------------------------------------- |
| `Onboarding`       | The primary record tracking a user's onboarding journey.                    |
| `Template`         | Blueprints for onboarding workflows based on roles.                         |
| `Task`             | Individual steps within a template.                                         |
| `TaskCompletion`   | Records of users completing specific tasks.                                 |
| `Document`         | Definitions of required documents (e.g., ID, Contract).                     |
| `DocumentUpload`   | The actual file uploads and their verification status.                      |
| `Training`         | Educational modules assigned to templates.                                  |
| `TrainingProgress` | Individual user progress and completion status for training.                |
| `Equipment`        | Asset requests associated with the onboarding process.                      |

### Equipment Property Reference

| Property       | Type      | Description                                                 |
| :------------- | :-------- | :---------------------------------------------------------- |
| `user_id`      | `int`     | The ID of the user.                                         |
| `request_type` | `string`  | Type of equipment (e.g., Laptop, Phone).                    |
| `status`       | `string`  | Status of the request (`pending`, `assigned`, `delivered`). |
| `asset_tag`    | `string?` | Unique asset identifier.                                    |
| `notes`        | `string?` | Optional notes.                                             |
