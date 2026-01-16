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
- Create necessary database tables (`onboard_*`).
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

### Task & Document Management

```php
// Mark task as complete
Onboard::completeTask($user, $slackSetupTask, 'Setup complete with help from IT.');

// Verify a document
Onboard::verifyDocument($passportUpload, $hrUser);
```

### Training

```php
// Update training progress
Onboard::training()->updateProgress($user, $safetyTraining, 'in_progress');
```

### Equipment Provisioning

Track asset requests like laptops, phones, and access badges.

```php
use Onboard\Models\Equipment;

// Request equipment
$equipment = Equipment::create([
    'user_id' => $user->id,
    'request_type' => 'Laptop',
    'status' => 'pending',
    'notes' => 'MacBook Pro 14"'
]);

// Update status
$equipment->update(['status' => 'delivered', 'asset_tag' => 'ASSET-123']);
```

## Analytics

Track onboarding health and individual progress.

```php
$stats = Onboard::analytics()->overview();
// Returns: ['total_onboarding' => 10, 'completed' => 5, 'in_progress' => 3, 'overdue' => 2]

$userProgress = Onboard::analytics()->progress($userId);
// Returns: (float) 75.5
```

## Integrations

- **Scout**: Automatically initiate onboarding when a candidate is hired.
- **Flow**: (Planned) Sync onboarding tasks with the global task management system.
- **Media**: Secure storage for all uploaded documents.
- **Audit**: Immutable trail of every completion and verification event.
- **Metric**: Automatically trigger goal initialization when onboarding is 100% complete.

## Models

### Equipment

The `Onboard\Models\Equipment` model represents a piece of equipment requested for a user.

| Property       | Type      | Description                                                 |
| :------------- | :-------- | :---------------------------------------------------------- |
| `user_id`      | `int`     | The ID of the user.                                         |
| `request_type` | `string`  | Type of equipment (e.g., Laptop, Phone).                    |
| `status`       | `string`  | Status of the request (`pending`, `assigned`, `delivered`). |
| `asset_tag`    | `string?` | Unique asset identifier.                                    |
| `notes`        | `string?` | Optional notes.                                             |
