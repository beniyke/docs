# Guide

The **Guide** package provides a robust FAQ and knowledge base system for the Anchor Framework. It features hierarchical categories, rich media support, full-text search, and helpfulness analytics.

## Core Capabilities

- **Hierarchical Categories**: Organize articles into nested categories.
- **Rich Media**: Attach images and videos to help articles.
- **Full-text Search**: Find articles quickly with built-in search analytics.
- **Helpfulness Feedback**: Collect user ratings and comments on articles.
- **Automatic Audit**: Logs article views and management actions.

## Installation

Guide is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Guide --packages
```

This command will:

- Publish the `guide.php` configuration file.
- Create necessary database tables (`guide_*`).
- Register the `GuideServiceProvider`.

## Basic Usage

### Creating Categories

```php
use Guide\Guide;

// Create a parent category
$billing = Guide::category()
    ->name('Billing & Payments')
    ->description('Everything related to your invoices and plans.')
    ->create();

// Create a sub-category
Guide::category()
    ->name('Refunds')
    ->parent($billing)
    ->create();
```

### Fetching Categories

```php
// Find by slug
$category = Guide::findCategory('billing-payments');

// Find by refid
$category = Guide::findCategoryByRefId('cat_abcdef123');
```

### Managing Articles

```php
// Create a published article
$article = Guide::article()
    ->title('How to Update Your Card')
    ->content('Go to Settings > Billing and click "Update Card"...')
    ->category($billing)
    ->status('published')
    ->create();
```

### Fetching Articles

```php
// Find by slug
$article = Guide::findArticle('how-to-update-your-card');

// Find by refid
$article = Guide::findArticleByRefId('art_xyz789');
```

### Searching Articles

```php
$results = Guide::search('payment');

foreach ($results as $article) {
    echo $article->title;
}
```

### Search with Filters & Metadata

```php
$results = Guide::search('refund', [
    'category' => $billing->id
], [
    'ip' => request()->ip(),
    'user_agent' => request()->header('User-Agent')
]);
```

## Analytics & Feedback

### Recording Views

```php
Guide::analytics()->recordView($article);
```

### Submitting Feedback

```php
Guide::analytics()->submitFeedback(
    $article,
    rating: 5,
    comment: 'Very helpful!'
);
```

### Popular Articles

```php
// Returns: [[Article Model], [Article Model], ...]
$popular = Guide::analytics()->getPopularArticles(limit: 5);
```

### Analytics Sample Data

The `Guide::analytics()` service provides insights into how users interact with your help center.

#### Search Logs (`SearchLog::class`)

| Property        | Type     | Description                              |
| :-------------- | :------- | :--------------------------------------- |
| `query`         | `string` | The search term entered by the user.     |
| `results_count` | `int`    | Number of articles found for this query. |
| `ip_address`    | `string` | The IP address of the user.              |
| `metadata`      | `json`   | Browser and session data.                |

#### Helpfulness Feedback (`Feedback::class`)

| Property           | Type     | Description                   |
| :----------------- | :------- | :---------------------------- |
| `guide_article_id` | `int`    | The ID of the rated article.  |
| `rating`           | `int`    | Rating value (typically 1-5). |
| `comment`          | `string` | Optional user feedback.       |

## Advanced Usage

### Article Relationships

You can link related articles to help users discover more content.

```php
Guide::relateArticles($article, $anotherArticle);
```

### Media Integration Types

Standardize how media is attached to articles for consistent rendering.

```php
// Attach as a main featured image
Guide::attachMedia($article, $mediaId, type: 'featured');

// Attach as a downloadable resource
Guide::attachMedia($article, $mediaId, type: 'attachment');
```

### Article Scopes

Standard Eloquent scopes for common queries:

```php
use Guide\Models\Article;

// Get all published articles
$articles = Article::published()->get();

// Get most viewed articles
$popular = Article::orderBy('view_count', 'desc')->limit(10)->get();
```

## Integrations

### Media Attachments

```php
// Assuming $mediaId from Media package
Guide::attachMedia($article, $mediaId, type: 'featured');
```

### Related Articles

```php
Guide::relateArticles($article, $anotherArticle);
```

## Configuration

Configuration is located in `App/Config/guide.php`:

```php
return [
    'search' => [
        'limit' => 10,
        'log_enabled' => true,
    ],
    'feedback' => [
        'enabled' => true,
    ],
];
```
