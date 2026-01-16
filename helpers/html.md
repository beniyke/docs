# HTML Helpers

The `Helpers\Html` namespace provides a set of tools for generating HTML elements, building reusable UI components, and managing static assets with automatic cache-busting.

> Use the global helpers `html()`, `component()`, and `assets()` for quick access.

## HtmlBuilder

The `HtmlBuilder` class offers a fluent interface for generating HTML strings. It handles attribute escaping and supports both standard and self-closing tags.

#### basic usage

```php
// Simple element
echo html()->element('span', 'Active', ['class' => 'badge'])->render();

// Nesting with start/close
echo html()->startTable(['class' => 'table'])->render();
echo html()->startRow()->render();
echo html()->dataCell('Row 1 Content')->render();
echo html()->closeRow()->render();
echo html()->closeTable()->render();
```

#### element / open

```php
element(string $tag, string $content = '', array $attributes = [], bool $self_closing = false): self

open(string $tag, array $attributes = []): self
```

`element` builds a complete tag. `open` seulement builds the opening tag (useful for manual nesting).

#### link / image

```php
link(string $content, string $href, array $attributes = []): self
image(string $src, string $alt = '', array $attributes = []): self
```

Convenient shortcuts for `<a>` and `<img>` tags.

#### Multimedia

```php
video(string $src, string $type = '', array $attributes = []): self
audio(string $src, array $attributes = []): self
```

Generates `<video>` and `<audio>` tags.

#### Form Elements

- `startForm(array $attributes = []): self`
- `closeForm(): self`
- `input(array $attributes = []): self`
- `textArea(string $content = '', array $attributes = []): self`
- `button(string $content, array $attributes = []): self`
- `label(string $content, array $attributes = []): self`
- `fieldset(string $content, array $attributes = []): self`
- `legend(string $content, array $attributes = []): self`
- `select(string $options, array $attributes = []): self`
- `options(array $options): self` (Helper for select options)
- `option(string $content, array $attributes = []): self`
- `optgroup(string $content, array $attributes = []): self`

#### Table Elements

- `startTable(array $attributes = []): self`
- `closeTable(): self`
- `startRow(array $attributes = []): self`
- `closeRow(): self`
- `headerCell(string $content, array $attributes = []): self`
- `dataCell(string $content, array $attributes = []): self`

#### Layout & Semantic Tags

- `div(string $content, array $attributes = []): self`
- `span(string $content, array $attributes = []): self`
- `paragraph(string $content, array $attributes = []): self`
- `section(string $content, array $attributes = []): self`
- `article(string $content, array $attributes = []): self`
- `header(string $content, array $attributes = []): self`
- `footer(string $content, array $attributes = []): self`
- `nav(string $content, array $attributes = []): self`
- `aside(string $content, array $attributes = []): self`

#### Assets & Metadata

- `script(string $content, array $attributes = []): self`
- `style(string $content, array $attributes = []): self`
- `linkTag(array $attributes = []): self` (for `<link>`)
- `meta(array $attributes = []): self`

#### Utilities

- `addRawHtml(string $html): self` - Append unescaped HTML.
- `render(): string` - Returns the generated HTML string and resets the buffer.
- `reset(): self` - Manually clear the internal buffer.

## HtmlComponent

`HtmlComponent` manages reusable UI fragments (templates). It automatically handles **old input** and **validation errors** from the `Flash` helper when a `name` attribute is present.

#### Usage

```php
echo component('input')
    ->attributes(['name' => 'email', 'type' => 'email'])
    ->render();
```

#### Methods

- `path(string $path): self` - Set a custom component file path.
- `data(array $data): self` - Pass shared variables to the component template.
- `content(mixed $content): self` - Set the primary value (accessible as `$value` in the template).
- `attributes(array $attributes): self` - Set HTML attributes.
- `options(array $options, bool $description = true): self` - Set data for select/radio components.
- `selected(mixed $selected): self` - Set the active value for options.
- `flagIf(bool $is_invalid): self` - Manually trigger error state (adds `is-invalid` class).

## Assets

The `Assets` class resolves paths to public files and automatically appends a modification timestamp for cache-busting.

> [!NOTE]
> Asset files should be placed in `public/assets/`.

#### url

```php
url(?string $file = null): string
```

Generates a fully qualified URL for a static asset.

- **Cache Busting**: If the file exists, its timestamp is appended (`?1712345678`), forcing browsers to reload the file after any update.
- **Smart Resolution**: Correctly handles absolute URLs, protocol-relative URLs (`//example.com`), and local paths.
- **Example**: `assets()->url('css/app.css')` returns `https://site.com/public/assets/css/app.css?1712345678`.

## Related

- [Form](form-helper.md) - Complex form management and validation
- [Flash](flash-helper.md) - Error messages and old input
- [URL](url-helper.md) - General URL generation
- [Paths](paths-helper.md) - Application path helpers
