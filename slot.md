# Slot

Slot is a scheduling engine that provides a flexible system for managing recurring events, availabilities, appointments, and time slots directly on your models. Whether you're building a clinic booking system, a resource rental platform, or a workforce management tool, Slot handles the complex geometry of time for you.

## Features

- **Polymorphic Scheduling**: Effortlessly add calendars to any model using the `HasSlots` trait.
- **Availability & Blocking**: Clearly define when resources are free and when they are strictly unavailable.
- **Extreme Fluency**: Configure complex periods and schedules using a natural, chainable PHP API.
- **Advanced Recurrence**: Native support for daily, weekly, monthly, and yearly repeating schedules.
- **Automatic Conflict Detection**: Prevent double-bookings with customizable, matrix-based overlap rules.
- **Smart Slot Generator**: Instantly calculate bookable time windows with configurable buffers and gaps.
- **Reminder System**: Integrated CLI command and notification events for upcoming bookings.

## Architecture

Slot uses a polymorphic architecture that decouples the "resource" being booked from the "reservation" itself.

### Core Concepts

- **Schedulable**: The owner of the calendar (the resource being booked).
- **SlotSchedule**: The ruleset for time. It defines windows of availability or pre-booked appointments.
- **SlotBooking**: An actual commitment of time by a bookable entity against a schedule.
- **Bookable**: The entity making the reservation (usually a User or Customer).
- **Period**: A fluent helper class for creating, manipulating, and comparing time ranges.

## Installation

### Install the Package

Install the Slot package via the CLI:

```bash
php dock package:install Slot --packages
```

### Database & Config

Running the installer automatically creates the `slot_*` tables and publishes the configuration to `App/Config/`.

## Getting Started

### Preparing Your Model

Apply the `HasSlots` trait to any class that should have scheduling capabilities.

```php
use Database\BaseModel;
use Slot\Traits\HasSlots;

class Doctor extends BaseModel
{
    use HasSlots;
}
```

### Defining Availability

Availabilities are the "open" windows in your calendar.

```php
use Slot\Period;

$doctor = Doctor::find(1);

// Monday 9 AM - 5 PM
$doctor->availability(
    Period::make('09:00', '17:00')->title('Standard Shift')
);
```

### Fetching Available Slots

Generate 30-minute bookable slots for a specific window:

```php
$timeWindow = Period::make('09:00', '17:00')->slotDuration(30);
$availableSlots = $doctor->getAvailableSlots($timeWindow);

foreach ($availableSlots as $slot) {
    echo "Time: " . $slot['starts_at']->format('H:i');
}
```

### Generating Slots

Divide a period into bookable chunks:

```php
$shift = Period::make('09:00', '17:00');

// Split shift into 30-minute slots with a 10-minute gap between them
$slots = $shift->split(30, 10);

foreach ($slots as $slot) {
    echo $slot->toString(); // e.g., "09:00 to 09:30 (30 minutes)"
}
```

## Use Case:

#### Shift Management

Managing a complex workforce requires more than just start and end times. Here is how to handle a nurse's shift with breaks and conflict prevention.

### The Shift Action

```php
namespace App\Actions;

use App\Models\Employee;
use Slot\Period;
use Slot\Slot;

class AssignShiftAction
{
    public function execute(Employee $employee, string $start, string $end, ?string $breakStart = null)
    {
        return DB::transaction(function() use ($employee, $start, $end, $breakStart) {
            // 1. Define the Main Shift
            $shift = Period::make($start, $end)
                ->title('Day Shift')
                ->asAvailability();

            // 2. Check for conflicts with existing assignments
            if (Slot::forModel($employee)->checkConflicts($shift)) {
                throw new \RuntimeException("Employee is already assigned during this period.");
            }

            // 3. Register the shift
            $schedule = Slot::forModel($employee)->availability($shift);

            // 4. Add a Mandatory Break (Blocked time)
            if ($breakStart) {
                $break = Period::make($breakStart, DateTimeHelper::parse($breakStart)->addMinutes(60))
                    ->title('Lunch Break')
                    ->asBlocked();

                Slot::forModel($employee)->blocked($break);
            }

            return $schedule;
        });
    }
}
```

### Implementation in a Scheduler UI

```php
public function assign(Request $request, AssignShiftAction $action)
{
    $nurse = Employee::find($request->employee_id);

    $action->execute(
        $nurse,
        $request->shift_start,
        $request->shift_end,
        $request->break_time
    );

    return response()->json(['message' => 'Shift assigned successfully']);
}
```

## Advanced Conflict Management

Slot uses a configurable matrix to determine if two periods can coexist.

### Overlap Rules

You can override default behavior fluently on the `Period` object:

```php
$meeting = Period::make('10:00', '11:00')
    ->asAppointment()
    ->allowBlocked()      // Can overlap with "Blocked" type
    ->denyAvailability(); // Strictly cannot overlap with "Availability"
```

## Reminder System

Slot includes a built-in reminder system to notify users of upcoming appointments.

### CLI Task

Run the following command (usually via cron every 5-15 minutes):

```bash
php dock slot:remind --minutes=30
```

### How it Works

- **Scanning**: The command finds all `confirmed` bookings starting in the next N minutes.
- **Notification**: For each booking, it automatically sends a `BookingReminderNotification` to the `bookable` entity.
- **Event**: Dispatches a `Slot\Events\BookingReminderEvent` so you can hook in custom logic (SMS, Push etc.).

## Analytics

The `Slot` package provides powerful tools for tracking bookings and generating calendar-compatible data.

```php
use Slot\Analytics;

// Get booking summary (confirmed, cancelled, completed)
$summary = Analytics::getBookingsSummary($from, $to);

// Get daily booking trends
$trends = Analytics::getDailyBookings($from, $to);

// Calculate occupancy rate for a specific model
$occupancy = Analytics::getOccupancyRate($user, $start, $end);

/**
 * Standardized Calendar Events
 *
 * Generates data ready for FullCalendar or other frontend libraries.
 */
$events = Analytics::getCalendarEvents($asset, $start, $end);
```

The `getCalendarEvents` method returns an array of events including availability (green) and bookings (blue), with integrated metadata.

## API Reference

### Period Builder (Fluent API)

| Method                                        | Description                            |
| :-------------------------------------------- | :------------------------------------- |
| `make($start, $end)`                          | Static initializer.                    |
| `daily/weekly/monthly($interval, $endsAt)`    | Configure recurrence rules.            |
| `title($string)`                              | Set a display label.                   |
| `asAvailability/Appointment/Blocked/Custom()` | Set the schedule type.                 |
| `slotDuration($mins)` / `gap($mins)`          | Configuration for slot generation.     |
| `buffer($before, $after)`                     | Adds "padding" time around the period. |
| `allowOverlap(...$types)`                     | Manually whitelist overlapping types.  |
| `denyOverlap(...$types)`                      | Manually blacklist overlapping types.  |
| `split($mins, $gap)`                          | Returns an array of Periods.           |
| `overlaps(Period $other)`                     | Boolean comparison.                    |

### Slot (Facade)

| Method                         | Description                                    |
| :----------------------------- | :--------------------------------------------- |
| `forModel(object $model)`      | Starts a fluent chain for a specific resource. |
| `checkConflicts(Period $p)`    | Returns array of conflicting SlotSchedules.    |
| `getAvailableSlots(Period $r)` | Returns generated time windows (array).        |
| `analytics()`                  | Returns the `SlotAnalytics` service instance.  |

### Analytics (Facade)

| Method                                 | Description                                                     |
| :------------------------------------- | :-------------------------------------------------------------- |
| `getBookingsSummary(?$f, ?$t)`         | Returns summary of confirmed, cancelled, and completed booking. |
| `getDailyBookings(string $f, $t)`      | Returns daily booking counts.                                   |
| `getCalendarEvents(object $m, $s, $e)` | Returns standardized events for calendar UI.                    |
| `getOccupancyRate(object $m, $s, $e)`  | Returns percentage of booked time vs available time.            |

### Models & Enums

- **SlotSchedule**: Persists the rules. Casts `type` to `ScheduleType`.
- **SlotBooking**: Persists the reservation. Casts `status` to `BookingStatus`.
- **ScheduleType**: `Availability`, `Appointment`, `Blocked`, `Custom`.
- **BookingStatus**: `Pending`, `Confirmed`, `Cancelled`, `Completed`.
 
## Automation
 
The Slot package automatically registers its reminder task in the framework scheduler. This ensures timely delivery of notifications to both hosts and guests.
 
```php
// packages/Slot/Schedules/BookingReminderSchedule.php
namespace Slot\Schedules;
 
use Cron\Interfaces\Schedulable;
use Cron\Schedule;
 
class BookingReminderSchedule implements Schedulable
{
    public function schedule(Schedule $schedule): void
    {
        $schedule->task()
            ->signature('slot:remind')
            ->everyThirtyMinutes();
    }
}
```

## Technical Summary

The Slot package provides a complete solution for scheduling. By utilizing **extreme fluency** and **automated conflict detection**, it allows developers to focus on their business logic while the engine handles the complexities of time, recurrence, and reminders.
