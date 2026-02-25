# Academy

**Academy** is a professional-grade Learning Management System (LMS) package for the Anchor Framework. It provides a robust engine for creating, managing, and monetizing educational content, from simple video courses to complex corporate training portals with automated assessments and certifications.

## Features

- **Hierarchical Content**: Organize learning into Programs, Modules, and Lessons.
- **Fluent Content Builder**: Create and structure programs using a developer-friendly fluent API.
- **Advanced Assessment Engine**: Build quizzes with multiple question types (MCQ, True/False, Essay) and automated grading.
- **Learner Lifecycle Management**: Handle enrolments, progress tracking, and milestone achievements.
- **Monetization & Instalments**: Sell individual courses or recurring subscription access via deep integration with **Pay**, **Wave**, and **Wallet**.
- **Gamification**: Reward learners with automated badges and rank-based achievements.
- **Certification**: Generate, issue, and verify secure PDF certificates upon program completion.
- **Community Learning**: Integrated discussion forums per program or lesson via the **Hub** package.
- **CLI Maintenance**: Dedicated commands for certificate verification, payment syncing, and data pruning.
- **Universal RefIDs**: Secure, public-facing identifiers for all Academy entities.

## Public Identifiers (RefID)

Academy uses a universal `refid` system for public-facing identifiers. This ensures that internal database IDs are never exposed in URLs or APIs, enhancing security and preventing enumeration attacks.

### Usage in Models

All Academy models implement the `HasRefid` trait and have a unique prefix (e.g., `prg_` for programs, `enr_` for enrolments).

```php
// Retrieve a program by its public refid
$program = AcademyProgram::where('refid', 'prg_abc123')->first();

// Generate a new refid manually if needed
$newRefid = AcademyProgram::generateRefid();
```

### Table Mapping Prefixes

| Entity          | Prefix | Example          |
| :-------------- | :----- | :--------------- |
| Program         | `prg_` | `prg_x8k2...`    |
| Module          | `mod_` | `mod_y1p9...`    |
| Lesson          | `les_` | `les_z4q0...`    |
| Enrolment       | `enr_` | `enr_a2b3...`    |
| Assessment      | `asm_` | `asm_c4d5...`    |
| Question        | `que_` | `que_e6f7...`    |
| Choice          | `cho_` | `cho_g8h9...`    |
| Submission      | `sub_` | `sub_i0j1...`    |
| Certificate     | `crt_` | `crt_k2l3...`    |
| Badge           | `bdg_` | `bdg_m4n5...`    |
| Rating          | `rat_` | `rat_p6q7...`    |
| Discussion      | `dsc_` | `dsc_r8s9...`    |
| Note            | `not_` | `not_t0u1...`    |
| Resource        | `res_` | `res_v2w3...`    |

## Installation

Academy is an optional package that requires installation.

### Install via Dock

```bash
php dock package:install Academy --packages
```

This command will:

- Publish the `App/Config/academy.php` configuration file.
- Register the `AcademyServiceProvider`.
- Run the required database migrations.
- Enable the global `academy()` helper and `Academy` facade.

### Configure Your LMS

Edit `App/Config/academy.php` to define your LMS defaults:

```php
return [
    'currency' => 'USD',
    
    // Feature Toggles (Deep Integrations)
    'integrations' => [
        'wave' => true,    // Subscriptions
        'wallet' => true,  // Credit-based learning
        'hub' => true,     // Course discussions
        'blish' => true,   // Newsletter sync
        'audit' => true,   // Activity logging
    ],

    // Certification Rules
    'certificates' => [
        'enabled' => true,
        'prefix' => 'CERT-',
        'template' => 'default',
    ],

    // Gamification
    'rewards' => [
        'enabled' => true,
        'points_per_lesson' => 10,
        'points_per_assessment' => 50,
    ],
];
```

## Architecture & Concepts

To build effectively with Academy, it's important to understand the hierarchy of entities and the service-oriented architecture.

### The Content Hierarchy

- **Programs**: The container for the entire course (e.g., "Fullstack Web Development").
- **Modules**: Logical groupings within a program (e.g., "Module 1: Introduction to PHP").
-**Lessons**: The actual learning units (Text, Video, or Quiz).
- **Assessments**: Quizzes or exams attached to modules or lessons to validate learning.

### The Learner Lifecycle

Everything in Academy revolves around the **Enrolment**. When a user joins a program, an `AcademyEnrolment` record is created. This record serves as the "source of truth" for:

- **Status**: (Pending, Active, Completed, Suspended)
- **Progress**: A percentage dynamically calculated based on completed lessons.
- **Milestones**: Tracked badges, grades, and eventual certification.

## Basic Usage

```php
use Academy\Academy;

$program = Academy::program()
    ->titled('Mastering the Anchor Framework')
    ->described('Become a professional developer with Anchor.')
    ->withContent('This program covers everything from core to advanced packages.')
    ->create();
```

> [!TIP]
> **Instructor Auto-Assignment**: When `create()` is called, the system automatically assigns the current authenticated user as an instructor if no `instructor_id` is provided in the data array.

### Enrolling a Learner

To enrol a user in a program, you can use the quick-access method on the facade:

```php
$enrolment = Academy::enrol($user->id, $program->id);

// Activate the enrolment (e.g., after payment confirmation)
Academy::enrolments()->activate($enrolment);
```

> [!NOTE]
> Upon enrolment, a unique **Admission Number** (e.g., `ADM-2026-0001`) is automatically generated and assigned to the enrolment record.

```php
Academy::progress()->completeLesson($enrolment->id, $lesson->id, [
    'time_spent' => 3600, // seconds
]);

$currentProgress = $enrolment->fresh()->progress_percent; // e.g., 25%
```

### Advanced Access Control (Drip & Prerequisites)

```php
// 1. Drip Content: Release Module 2 after 7 days of enrolment
$module->update(['metadata' => ['drip_delay' => 7]]);

// 2. Prerequisites: Lesson B requires Lesson A to be completed
$lessonB->update(['metadata' => ['prerequisite_lesson_id' => $lessonA->id]]);

// 3. Validation
$canAccess = Academy::programs()->canAccess($userId, $lessonB->id); // returns false if Lesson A is not completed

### Data Aggregation & Deep Fetching

Academy provides "Deep Fetching" methods to retrieve a full tree of data for common entities. All methods support both integer IDs and string `refid`s.

| Method | Returns | Included Associations |
| :--- | :--- | :--- |
| `getProgramDetails(id)` | `array` | Modules, Lessons, Instructors, Resources, Announcements, Plans. |
| `getLessonDetails(id)` | `array` | Parent Module/Program, Resources, Assessment, Live Session. |
| `getEnrolmentDetails(id)`| `array` | Program Tree, Progress, Payment Instalments, Certificate, Notes. |

#### Example Usage

```php
// Fetching a Program via RefID
$details = Academy::getProgramDetails('prg_abc123');

// Accessing nested data
$modules = $details['modules'];
$instructors = $details['instructors'];
```

### Instructor & Restricted Access

- **Instructor Override**: Any user assigned as an instructor to a program via `addMember()` has full access to all lessons, bypassing drip-delay and prerequisites.
- **Specific Learner Access**: You can restrict a lesson to specific users by adding an `allowed_user_ids` array to the lesson's `metadata`.
- **Multiple Instructors**: Programs support multiple instructors. If no `instructor_ids` are provided during `create()`, the creator is assigned as the primary instructor by default.

#### Implementation Example

```php
// 1. Fluent Builder (Multiple Instructors)
$program = Academy::program()
    ->titled('Advanced Laravel Architecture')
    ->withInstructors([$leadId, $assistantId])
    ->create();

// 2. Manual Assignment (Program Service)
Academy::programs()->addMember($program->id, $user->id, 'instructor');

// 3. Metadata Restrictions (per Lesson)
$lesson->update(['metadata' => ['allowed_user_ids' => [1, 5, 8]]]);
```

## Assessment Engine

The Assessment Engine allows you to create complex quizzes and exams with automated feedback loops.

### Creating an Assessment

Assessments can be attached to modules or lessons.

```php
$assessment = Academy::assessments()->create([
    'title' => 'Core Architecture Quiz',
    'passing_score' => 80, // percentage
    'attempts_allowed' => 3,
]);
```

### Building Questions & Choices

```php
$question = Academy::assessments()->addQuestion($assessment->id, [
    'content' => 'What is the entry point of an Anchor request?',
    'type' => 'mcq',
    'points' => 10,
]);

Academy::assessments()->addChoice($question->id, 'index.php', isCorrect: true);
Academy::assessments()->addChoice($question->id, 'kernel.php', isCorrect: false);
```

### The Attempt Lifecycle

```php
// 1. Start an attempt
$submission = Academy::assessments()->startAttempt($assessment->id, $enrolment->id);

// 2. Process submission (from a DTO or Request)
Academy::assessments()->submit($submission, [
    $question1->id => ['choice_id' => $choice1->id],
    $question2->id => ['choice_id' => $choice2->id],
]);

// 3. Check results
if ($submission->is_passing) {
    echo "Learner passed with " . $submission->percent_score . "%";
}
```

### Bulk Assessment Operations

For high-volume content creation and grading, Academy supports bulk operations.

#### Bulk Question Creation

Use `bulkAddQuestions` to efficiently create multiple questions and their respective choices in a single transaction.

```php
Academy::assessments()->bulkAddQuestions($assessmentId, [
    [
        'text' => 'Question 1',
        'type' => 'mcq',
        'points' => 5,
        'choices' => [
            ['text' => 'Choice 1', 'is_correct' => true],
            ['text' => 'Choice 2', 'is_correct' => false],
        ],
    ],
    [
        'text' => 'Question 2',
        'type' => 'mcq',
        'points' => 10,
        'choices' => [...],
    ],
]);
```

#### Bulk Submission

The `submit` method natively accepts an array of answers, allowing for a single-request submission of an entire assessment.

```php
Academy::assessments()->submit($submission, [
    $q1_id => ['choice_id' => $c1_id],
    $q2_id => ['choice_id' => $c2_id, 'content' => 'Optional essay text'],
    $q3_id => ['file_path' => 'path/to/upload.jpg'], // For file-upload types
]);
```

#### Bulk Manual Grading

For essay-heavy programs, you can batch process grades.

```php
$grades = [
    ['submission_id' => 1, 'score' => 85, 'feedback' => 'Great job!'],
    ['submission_id' => 2, 'score' => 45, 'feedback' => 'Please rewrite.']
];

foreach ($grades as $g) {
    $submission = AcademySubmission::find($g['submission_id']);
    Academy::assessments()->manualGrade($submission, $g['score'], $g['feedback']);
}
```

## Monetization & Integrations

Academy thrives by leveraging the Anchor ecosystem to handle complex financial logic.

### Payment Plans & Instalments (Pay Package)

You can define payment plans that allow learners to pay in instalments.

```php
$plan = Academy::payments()->createPlan([
    'name' => 'Extended BootCamp',
    'price' => 120000, // $1,200.00
    'instalment_count' => 3,
    'instalment_interval' => 30, // days
]);

// Initialize instalments for a new enrolment
Academy::payments()->initializeInstalments($enrolment);
```

### Subscription Gating (Wave Package)

Gate entire programs behind recurring subscription plans.

```php
$program = Academy::program()
    ->titled('VIP Learning Pass')
    ->create();

// Gating via metadata (Wave looks for these Plan IDs during enrolment)
$program->update(['metadata' => ['required_plan_ids' => ['pro-monthly', 'enterprise']]]);
```

### Credit-Based Learning (Wallet Package)

Allow learners to deposit funds and pay for course milestones using their internal balance.

```php
// Pay an instalment specifically via wallet
Academy::payments()->initializePayment($instalment, driver: 'wallet');

// Deposit funds to wallet for academy use
Academy::payments()->depositToWallet($user, 50000); // $500.00
```

## Gamification & Certification

Keep learners engaged with rewards and professional credentials.

### Badges & Achievements

Automate badge awards based on learning triggers.

```php
// Award a badge manually
Academy::badges()->award($user->id, $badge->id, $program->id);

// Or via triggers in App/Config/academy.php
'triggers' => [
    'lesson_completed' => ['Fast Learner Badge'],
    'program_completed' => ['Anchor Certified Pro'],
],
```

### Program Ratings & Reviews

Learners can rate programs and provide feedback to help others.

```php
// Submit a rating
Academy::ratings()->submit($userId, $programId, rating: 5, review: 'Excellent content!');

// Get average rating
$average = Academy::ratings()->getAverageRating($programId);

// Get featured / top reviews
$reviews = Academy::ratings()->getFeaturedReviews($programId, limit: 10);
```

### Secure Certification

Academy generates secure, verifiable credentials.

```php
// Issue certificate upon completion
$certificate = Academy::certificates()->issue($enrolment);

// Get a secure, signed sharing URL (Linked to the Link package)
$shareUrl = Academy::certificates()->getSharingUrl($certificate);
```

## Landing Pages & SEO

Academy automatically manages public-facing landing pages for your programs, optimized for conversion and search engines.

### Program Landing Pages

Landing pages are dynamically generated based on the program's slug and status.

```php
// Get comprehensive data for a landing page
$pageData = Academy::landingPages()->getPageData('mastering-anchor');

// Returns:
// [
//    'program' => (AcademyProgram),
//    'instructors' => (Collection of staff),
//    'modules_count' => 12,
//    'lessons_count' => 45
// ]
```

### SEO Integration (Rank)

If the **Rank** package is installed and enabled in `academy.php`, SEO metadata (Title, Description, Graph Images) is automatically injected into the page state when `getPageData()` is called.

## Live Learning & Attendance

Beyond static content, Academy supports synchronized live sessions for interactive teaching.

### Scheduling Live Sessions

Sessions can be linked to lessons of type `live`.

```php
$session = Academy::liveSessions()->schedule([
    'lesson_id' => $liveLesson->id,
    'provider' => 'zoom',
    'meeting_url' => 'https://zoom.us/j/123456789',
    'starts_at' => '2026-03-01 10:00:00',
]);
```

### Attendance Tracking

Track when learners join and leave live sessions to calculate engagement duration.

```php
// Record when a learner joins
Academy::liveSessions()->recordAttendance($session->id, $enrolment->id);

// Record when they leave (auto-calculates duration)
Academy::liveSessions()->recordLeave($session->id, $enrolment->id);
```

## Community Discussions

Engagement is boosted through integrated discussions, either standalone or linked to the **Hub** community package.

### Posting & Replying

```php
// Post a new topic
$topic = Academy::discussions()->post([
    'program_id' => $program->id,
    'user_id' => $user->id,
    'content' => 'How do I handle dependency injection?',
]);

// Reply to a topic
Academy::discussions()->post([
    'parent_id' => $topic->id,
    'user_id' => $admin->id,
    'content' => 'Use the resolve() helper!',
]);
```

### Management

Pin important discussions or resolve support-style queries.

```php
Academy::discussions()->pin($topic);
Academy::discussions()->resolve($topic);
```

## Transcripts & Reporting

Academy provides powerful reporting tools for learners and educators to track performance and results.

### Generating Transcripts
Transcripts provide a consolidated record of all assessments and grades for an enrolment.

```php
$transcript = Academy::reports()->getTranscript($enrolmentId);

// Sample Data Structure:
// [
//    'learner_name' => 'John Doe',
//    'program_title' => 'Mastering Anchor',
//    'status' => 'active',
//    'progress' => 45,
//    'assessments' => [
//        [
//            'title' => 'Core Quiz',
//            'score' => 95,
//            'is_passing' => true,
//            'submitted_at' => '2026-02-20',
//            'is_late' => false
//        ]
//    ]
// ]
```

### Student Performance Timeline
Visualize a learner's performance over time.

```php
$performance = Academy::analytics()->getLearnerPerformance($enrolmentId);

// Sample Data Structure:
// [
//    'labels' => ['02-15', '02-18', '02-20'], // graded_at dates
//    'values' => [85, 90, 75] // percent scores
// ]
```

### Detailed Progress Reports
Break down lesson completion with historical logs.

```php
$report = Academy::reports()->getProgressReport($enrolmentId);

// Sample Data Structure:
// [
//    'total_lessons' => 20,
//    'completed_count' => 9,
//    'percentage' => 45,
//    'lessons' => [
//        ['title' => 'Intro to PHP', 'completed_at' => '2026-02-10 14:00', 'time_spent' => 1200],
//        ['title' => 'Data Structures', 'completed_at' => '2026-02-12 10:30', 'time_spent' => 3600]
//    ]
// ]
```

### Lifecycle History
A unified log of all engagement events.

```php
$history = Academy::reports()->getLifecycleHistory($enrolmentId);

// Sample Data Structure:
// [
//    ['type' => 'lesson', 'event' => 'Completed lesson: Intro', 'date' => '2026-02-10'],
//    ['type' => 'assessment', 'event' => 'Submitted assessment: Core Quiz', 'date' => '2026-02-12', 'score' => 95]
// ]
```

## Performance & Analytics

Monitor program health and revenue via the analytics service.

```php
$metrics = Academy::analytics()->getProgramMetrics($program->id);

// Sample Data Structure:
// [
//    'total_enrolments' => 1250,
//    'completion_rate' => 68,
//    'average_progress' => 74,
//    'active_students' => 450
// ]
```

```php
// Trends for 'enrolments', 'revenue', or 'submissions'
$trends = Academy::analytics()->getHistory('enrolments', '30d');

// Sample Data Structure:
// [
//    'labels' => ['Feb 01', 'Feb 02', ...],
//    'values' => [12, 15, 8, 20, ...]
// ]
```

### Smart & Semantic Search V2

Academy provides a powerful deep-search engine that matches queries against programs, contents, resources, and announcements.

```php
// 1. Deep Search across all entities
$results = Academy::programs()->search('Database Optimization'); 
// Matches program titles, lesson content, resource titles, and announcements.

// 2. Filtered Search (e.g., only featured programs)
$featured = Academy::programs()->search('PHP', ['is_featured' => true]);

// 3. Dedicated Entity Search
$instructors = Academy::programs()->searchInstructors('John');
$learners = Academy::enrolments()->searchLearners('ADM-2026');
```

### Leaderboards & Ratings

Engage learners and showcase program quality with internal social metrics.

```php
// Get top 10 learners by progress
$leaderboard = Academy::analytics()->getLeaderboard($programId, limit: 10);

// Get a summary of all ratings for a program
$ratings = Academy::analytics()->getRatingSummary($programId);

// Sample Ratings Summary:
// [
//    'average' => 4.5,
//    'total' => 150,
//    'breakdown' => [5 => 100, 4 => 30, 3 => 10, 2 => 5, 1 => 5]
// ]
```

### Learner Notes & Wishlists

Learners can take personal notes on specific lessons or save programs for later.

#### Notes
```php
// Add a note
Academy::progress()->saveNote($enrolmentId, $lessonId, 'This is an important concept!');

// Retrieve all notes for an enrolment
$allNotes = Academy::progress()->getNotes($enrolmentId);
```

#### Wishlists
```php
// Add to wishlist
Academy::enrolments()->addToWishlist($userId, $programId);

// Get user wishlist
$wishlist = Academy::enrolments()->getWishlist($userId);
```

## Announcements & Engagement

Academy includes built-in tools for keeping learners informed and engaged.

### Program Announcements
Send broadcast messages to all enrolled learners.

```php
$announcement = AcademyAnnouncement::create([
    'program_id' => $program->id,
    'user_id' => $instructor->id,
    'title' => 'New Module Released!',
    'content' => 'Check out the new module on Advanced SQL.',
    'send_email' => true, // Triggers background email delivery
]);
```

### Discounts & Coupons (Wave Integration)
Discounts are handled via the **Wave** package. You can create coupons and apply them during the checkout/enrolment process.

```php
use Wave\Wave;

// Create a 20% discount coupon
$coupon = Wave::coupons()->create([
    'code' => 'ACADEMY20',
    'percent_off' => 20,
    'duration' => 'once',
]);
```

### Images & Media

Images in questions and options are typically handled by:

- **Embedding HTML**: Standard `<img>` tags in the `text` field.
- **Media Library**: Using Anchor's internal media service to attach assets to `AcademyQuestion` records.

## Resources & Documentation

Academy allows attaching external files, videos, and links to lessons as supplemental material.

### Attaching Resources

Resources are linked to specific lessons and can be managed via the `AcademyResource` model.

```php
use Academy\Models\AcademyResource;
use Academy\Enums\ResourceType;
use Academy\Enums\VideoProvider;

$resource = AcademyResource::create([
    'lesson_id' => $lesson->id,
    'title' => 'Advanced SQL Cheat Sheet',
    'type' => ResourceType::FILE,
    'path' => 'uploads/docs/sql-cheat-sheet.pdf',
]);

// For external videos
$video = AcademyResource::create([
    'lesson_id' => $lesson->id,
    'title' => 'Database Optimization',
    'type' => ResourceType::VIDEO_EXTERNAL,
    'provider' => VideoProvider::YOUTUBE,
    'path' => 'https://youtube.com/watch?v=123',
]);
```

- `file`: Locally uploaded documents or media.
- `video_external`: Videos hosted on providers like YouTube, Vimeo, Wistia, or Bunny.
- `link`: Direct URLs to external websites or documents.
- `embed`: Raw HTML/Iframe embed codes.

### Fetching Resources

```php
// Get all resources for a specific program
$programResources = Academy::programs()->getResources($programId);

// Get resources specifically attached to a lesson
$lessonResources = Academy::programs()->getResources(lessonId: $lessonId);
```

## Extensions & Late Policies

Manage deadlines and flexible learning paths.

### Enrolment Extensions

```php
Academy::enrolments()->extend($enrolmentId, 10); // Extend access by 10 days
```

### Bulk Enrollments

For cohort imports or enterprise onboarding, use the optimal `bulkEnrol` method.

```php
$userIds = [101, 102, 103];
Academy::enrolments()->bulkEnrol($userIds, $programId);
```

### Assessment Extensions

```php
Academy::assessments()->grantExtension($submission, '2026-03-30 23:59:59');
```

## CLI & Automation

Academy includes robust maintenance tools to handle background tasks.

| Command                     | Description                                              |
| :-------------------------- | :------------------------------------------------------- |
| `academy:verify-cert {num}` | Verify a learner's certificate manually.                |
| `academy:credentials:issue` | Bulk issue credentials to learners who completed courses. |
| `academy:payments:sync`     | Pull payment statuses from external gateways.            |
| `academy:prune:expired`     | Clean up expired enrolments and stale waitlists.        |

**Production Tip**: Automate these tasks via Cron.

```cron
0 0 * * * php dock academy:prune:expired
0 * * * * php dock academy:payments:sync
```

## Architecture Deep-Dive

### Learning Models

Academy is designed primarily as a **Self-Paced, Duration-Based** system.

- **Self-Paced**: Learners can enrol at any time and progress at their own speed.
- **Duration-Based**: Access is controlled via the `expires_at` property on the enrolment. If a program is part of a subscription, Wave manages the "heartbeat" of this access.

**Tip**: For **Cohort/Batch-based** learning, use the `metadata` column on a Program to store `start_date` and `end_date`, and use a custom middleware to gate access based on these dates.

## API Reference

### Academy Facade

| Method           | Returns                   | Description                               |
| :--------------- | :------------------------ | :---------------------------------------- |
| `program()`      | `ProgramBuilder`          | Start a fluent program definition.        |
| `enrol(...)`     | `AcademyEnrolment`        | Quick enrolment shortcut.                 |
| `gradeQuiz(...)` | `void`                    | Auto-grade an MCQ submission.             |
| `programs()`     | `ProgramManagerService`   | Access core program management.           |
| `enrolments()`   | `EnrolmentManagerService` | Access learner lifecycle methods.         |
| `assessments()`  | `AssessmentService`       | Access quiz/exam engine.                  |
| `analytics()`    | `AcademyAnalyticsService` | Access metrics and trends.                |
| `discussions()`  | `DiscussionService`       | Access community threads.                 |
| `liveSessions()` | `LiveSessionService`      | Manage Zoom/Meet sessions.                |
| `ratings()`      | `RatingService`           | Access program reviews and ratings.       |
| `badges()`       | `BadgeService`            | Manage learner achievements.              |
| `landingPages()` | `LandingPageService`      | Manage SEO-optimized landing pages.       |
| `reports()`      | `ReportingService`        | Access transcripts and detailed reports.  |

### EnrolmentManagerService

- `enrol(userId, programId, ?planId, ?refCode)`
- `bulkEnrol(userIds, programId, ?planId)`
- `activate(enrolment)`
- `isEnrolled(userId, programId)`
- `bulkIssueCredentials(?programId)`
- `pruneExpiredEnrolments()`
- `pruneExpiredWaitlists()`
- `extend(enrolment, days)`
- `getWishlist(userId)`
- `addToWishlist(userId, programId)`
- `searchLearners(query)`: Find students by Name, Email, or Admission ID.
- `generateAdmissionNumber(enrolmentId)`: Generates a unique `ADM-2026-0001` format ID and saves it to the `admission_id` column. (Note: This is called automatically during `enrol()`).

### PaymentManagerService

- `initializeInstalments(enrolment)`
- `initializePayment(instalment, ?driver)`
- `processPayment(enrolment, reference, amount)`
- `syncExternalPayments()`
- `getDefaulters(?programId)`: Returns overdue pending instalments.
- `getOutstandingBalance(enrolmentId)`: Total remaining debt.
- `getBalance(enrolmentId)`: Alias for `getOutstandingBalance`.
- `getOverdue(enrolmentId)`: Returns specific overdue items.
- `depositToWallet(user, amount)`: Fund learner wallet for academy use.

### ProgramManagerService

- `create(data)`
- `update(program, data)`
- `addMember(program, userId, role)`
- `publish(program)`
- `getMembers(programId, ?role)`
- `getResources(?programId, ?lessonId)`
- `getAnnouncements(programId)`
- `getDripContent(programId, enrolmentId)`
- `canAccess(userId, lessonId)`: Checks for Drip Content and Prerequisites.
- `search(query, filters)`: Matches Programs, Modules, Lessons, Resources, and Announcements.
- `searchInstructors(query)`: Find instructors by name or email.

### ProgressTrackingService

- `completeLesson(enrolmentId, lessonId, ?time)`
- `saveNote(enrolmentId, lessonId, content)`
- `getNotes(enrolmentId, ?lessonId)`
- `updateOverallProgress(enrolmentId)`: Force a recalculation of progress.

### AssessmentService

- `startAttempt(assessmentId, enrolmentId)`
- `submit(submission, answers)`
- `autoGrade(submission)`
- `manualGrade(submission, score, feedback)`
- `grantExtension(submission, date)`: Grant more time for an assessment.
- `canTake(userId, assessmentId)`: Check attempt limits and prerequisites.

- `getAssessmentLeaderboard(assessmentId, ?limit)`
- `getRatingSummary(programId)`
- `getHistory(metric, range)`

## Permissions & Authorization

Academy provides granular authorization checks for various user roles.

| Method | Role | Description |
| :--- | :--- | :--- |
| `canView(userId, programId)` | Learner/Instructor | Check if user can see program contents. |
| `canAccess(userId, lessonId)` | Learner/Instructor | Check if user can take a specific lesson (Drip/Prereq). |
| `canManage(userId, programId)` | Instructor | Check if user can edit or moderate a program. |
| `canTake(userId, assessmentId)` | Learner/Instructor | Check if user can start an assessment. |
| `canGrade(userId, submissionId)` | Instructor | Check if user can grade a specific submission. |

### Usage Examples

```php
// Gating a Management UI
if (Academy::programs()->canManage($user->id, $programId)) {
    // Show Edit/Grade buttons
}

// Starting an Assessment
if (Academy::assessments()->canTake($user->id, $assessmentId)) {
    return Academy::assessments()->startAttempt($assessmentId, $enrolmentId);
}
```
