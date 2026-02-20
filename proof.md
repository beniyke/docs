# Proof

The **Proof** package provides a comprehensive structure for managing social proof, including testimonials and case studies, to build trust and credibility.

## Installation

Proof is a **package** that requires module installation.

### Install the Package

```bash
php dock package:install Proof --packages
```

This command will:

- Publish the `proof.php` configuration file.
- Run the migration for Proof tables.
- Register the `ProofServiceProvider`.

### Configuration

Configuration file: `App/Config/proof.php`

## Core Concepts

### Testimonials

Customer quotes, ratings, and video testimonials.

### Case Studies

In-depth success stories with multiple sections (Problem, Solution, Results) and measurable metrics.

### Sources

The entities (people or companies) that provide the social proof.

## Usage

### Managing Testimonials

#### Creating a Testimonial (Fluent API)

```php
use Proof\Proof;

$testimonial = Proof::testimonial()
    ->source($sourceId)
    ->content('Anchor has transformed our workflow!')
    ->rating(5)
    ->verified()
    ->save();
```

#### Approving a Testimonial

```php
Proof::approve($testimonialId);
```

### Managing Case Studies

#### Creating a Case Study

```php
$caseStudy = Proof::caseStudy()
    ->source($sourceId)
    ->title('Scaling with Anchor')
    ->slug('scaling-with-anchor')
    ->summary('How we achieved 300% growth.')
    ->save();

// Adding sections
$caseStudy->sections()->create([
    'title' => 'The Challenge',
    'content' => 'We were struggling with legacy systems...',
    'order' => 1
]);

// Adding metrics
$caseStudy->metrics()->create([
    'label' => 'Growth',
    'value' => '300',
    'suffix' => '%'
]);
```

### Integrations

#### Form Integration (Stack)

Automatically convert form submissions into testimonials.

```php
use Proof\Proof;
use Stack\Models\Submission;

Proof::fromSubmission($submission, [
    'content' => 'feedback_text',
    'rating' => 'star_rating',
    'name' => 'customer_name'
]);
```

#### Secure Collection Requests

Generate secure, expiring links for customers to submit their proof.

```php
$request = Proof::request($source);
$url = Proof::collectionUrl($request);
```

#### Media Integration

Attach photos or videos from the Media package.

```php
Proof::attachMedia($testimonial, $mediaId, 'photo');
```

## Advanced Usage

### Approval Workflow

Proof includes a built-in state machine for testimonial approvals, integrated with the **Workflow** package.

```php
use Proof\Proof;

// Start the approval process
Proof::startApprovalWorkflow($testimonial);

// Signals (typically from an admin dashboard)
Proof::approve($testimonial->id);
Proof::reject($testimonial->id);
```

### Stack Submission Mapping

Map form fields to testimonial properties for automatic conversion.

```php
Proof::fromSubmission($submission, [
    'name' => 'customer_name',
    'email' => 'customer_email',
    'company' => 'business_name',
    'content' => 'feedback_message',
    'rating' => 'stars',
]);
```

## Analytics & Monitoring

Track social proof engagement.

### Recording Interactions

```php
Proof::analytics()->recordView($testimonial, ['ip' => request()->ip()]);
```

### Metrics Data Structure

The `Proof::analytics()->getMetrics()` method returns an array of engagement data.

| Metric   | Description                                       |
| :------- | :------------------------------------------------ |
| `views`  | Total number of times the proof was displayed.    |
| `clicks` | Number of clicks on "Learn More" or source links. |

## Configuration

Available in `App/Config/proof.php`:

- `form_integration`: Enable/disable automatic form conversion.
- `approval.required`: Whether testimonials need approval before display.
- `request.expiry_days`: Duration for secure collection links.
