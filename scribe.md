# Scribe

Scribe is a professional blogging and content management package for the Anchor Framework. It provides a robust foundation for building feature-rich blogs with categories, tags, comments, and built-in SEO controls.

## Features

- **Fluent Post Building**: Expressive API for creating, updating, and scheduling posts.
- **Taxonomies**: Nested Categories and flat Tags for organized content.
- **SEO & Social Metadata**: Built-in support for Meta Titles, Descriptions, and Open Graph tags.
- **Publishing Workflows**: Drafts, Scheduled, and Published statuses.
- **Analytics**: Track post views and engagement trends.
- **Defensive Integration**: Gracefully works with or without optional packages like `Audit` and `Media`.

## Installation

Scribe is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Scribe --packages
```

This command will:

- Publish the `scribe.php` configuration file.
- Run the migration for Scribe tables.
- Register the `ScribeServiceProvider`.

## Basic Usage

### Creating a Post

Use the `Scribe` facade to build a new post programmatically.

```php
use Scribe\Scribe;

$post = Scribe::post()
    ->title('The Future of Agentic Coding')
    ->content('Content goes here...')
    ->excerpt('A brief summary of the post.')
    ->category(5)
    ->tags([1, 2, 8])
    ->seo([
        'title' => 'Agentic Coding | Anchor Framework',
        'description' => 'Learn how AI agents are transforming software development.',
    ])
    ->status('published')
    ->create();
```

### Fetching Posts

```php
// Find by slug
$post = Scribe::findPost('the-future-of-agentic-coding');

// Find by refid
$post = Scribe::findPostByRefId('pst_abc123');
```

### Scheduling a Post

You can schedule posts for future publication using the `schedule` method.

```php
use Scribe\Scribe;
use Helpers\DateTimeHelper;

$post = Scribe::post()
    ->title('Upcoming Feature Announcement')
    ->content('Stay tuned...')
    ->status('scheduled')
    ->publishedAt(DateTimeHelper::now()->addDays(7))
    ->create();
```

## Category Management

Scribe supports hierarchical categories. Use the `CategoryBuilder` for fluent creation.

```php
use Scribe\Scribe;

$category = Scribe::category()
    ->name('Technology')
    ->description('Latest in tech.')
    ->create();

// Create a sub-category
$subCategory = Scribe::category()
    ->name('AI')
    ->parent($category)
    ->create();
```

### Fetching Categories

```php
// Find by slug
$category = Scribe::findCategory('technology');

// Find by refid
$category = Scribe::findCategoryByRefId('cat_xyz789');
```

## Advanced Usage

### Handling Comments

Scribe includes a simple comment system that can be moderated.

```php
use Scribe\Scribe;

$comment = Scribe::addComment($post, [
    'content' => 'Great article!'
], userId: 1);
```

### SEO Metadata Generation

If you don't manually set SEO meta, Scribe can generate it for you.

```php
use Scribe\Scribe;

$meta = Scribe::generateSeoMeta($post);
```

### Analytics Trends

Track engagement over time for specific posts.

```php
use Scribe\Scribe;

// Record a view
Scribe::recordView($post, $userId, $sessionId);

// Get view counts for the last 30 days
// Returns: [['date' => '2026-01-01', 'count' => 10], ...]
$trends = Scribe::analytics()->getPostTrends($post, days: 30);

// Get popular posts
// Returns: [[Post Model], [Post Model], ...]
$topPosts = Scribe::analytics()->getTopPosts(limit: 5);
```

## Integration

- **Media**: Use the `Media` facade to handle featured images and post assets via `attachMedia()`.
- **Audit**: Automatically logs publishing and scheduling events if installed.
- **Link**: Generate signed URLs for private or early-access post previews.
