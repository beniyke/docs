# Rank

Rank is a high-performance SEO management package for the Anchor Framework. It provides a fluent, expressive API for managing metadata, Open Graph tags, and Twitter Cards, with deep, zero-config integration into the **Route Context** system.

## Features

- **Fluent Metadata**: Chainable API for setting titles, descriptions, and keywords.
- **Context-Aware Titles**: Automatically generates human-readable page titles based on route `entity` and `action` context.
- **Social Media Ready**: Built-in support for Open Graph and Twitter Cards with smart fallbacks.
- **Canonical Handling**: Simple management of canonical URLs to prevent duplicate content issues.
- **Zero-Config Integration**: Automatically hooks into the `ViewEngine` for seamless template usage.
- **Custom Meta Tags**: Ability to render any arbitrary meta name/property pair.

## Installation

Rank is a **package** that requires installation before use.

### Install the Package

```bash
php dock package:install Rank --packages
```

This will automatically:

- Register the `RankServiceProvider`.
- Publish the `rank.php` configuration file.
- Register the `rank()` macro on the `ViewEngine`.
- Register the global `rank()` helper.

## Configuration

Configuration file: `App/Config/rank.php`

```php
return [
    'defaults' => [
        'title' => 'Anchor Framework',
        'description' => 'A powerful, lightweight PHP framework.',
        'image' => 'https://example.com/default-share.jpg',
        'type' => 'website',
    ],

    'og' => [
        'site_name' => 'Anchor App',
    ],

    'twitter' => [
        'card' => 'summary_large_image',
        'site' => '@anchorphp',
    ],
];
```

## Basic Usage

### In Templates

Access Rank directly via the `rank()` helper in any view:

```php
<!-- In your layout.php -->
<head>
    <?= $this->rank()->render() ?>
</head>
```

### Manual Overrides

You can override defaults or set specific values from your controller:

```php
// In a Controller
public function show(int $id): Response
{
    $product = Product::find($id);

    rank()
        ->setTitle($product->name)
        ->setDescription($product->short_description)
        ->setImage($product->featured_image);

    return $this->asView('details', compact('product'));
}
```

## Smart Features

### Automated Route Titles

Rank leverages the framework's automatic **Route Context** hydration to provide relevant titles without any manual intervention.

When a request is handled, the framework automatically extracts context from the Controller and Method:
- `App\Account\Controllers\UserController@edit`
- **Entity**: `User`
- **Action**: `edit`
- **Resulting Title**: "Edit User"

This automation ensures that your application has sensible, SEO-friendly titles out of the box for every module.

## Service API Reference

### Rank (Facade/Helper)

| Method | Description |
| :--- | :--- |
| `setTitle(string $title)` | Sets the page title and `og:title`. |
| `setDescription(string $description)` | Sets the meta description and `og:description`. |
| `setImage(string $url)` | Sets the share image for OG and Twitter. |
| `setCanonical(string $url)` | Sets the canonical URL. |
| `setMeta(string $name, string $content)` | Sets a standard `<meta name="...">`. |
| `setOg(string $property, string $content)` | Sets an Open Graph `<meta property="...">`. |
| `render()` | Generates the HTML string of all meta tags. |

## Security & Best Practices

- **Contextual Accuracy**: Ensure your Controller names are human-friendly as they are used directly in automated titles.
- **Image URLs**: Always use absolute URLs for social share images (`setImage`) to ensure platforms can crawl them correctly.
- **Macro Avoidance**: While Rank uses `Macroable`, avoid overwriting core `ViewEngine` methods to maintain framework stability.
