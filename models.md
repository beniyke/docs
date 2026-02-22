# Models (ORM)

Anchor includes a lightweight ORM that makes working with database tables intuitive. Each database table has a corresponding "Model" that is used to interact with that table.

## Defining Models

Models reside in `App/src/{Module}/Models/` and extend `Database\BaseModel`.

> **Singular table names**: The framework uses singular table names by convention (e.g., `user` for `User`, `post` for `Post`). The `$table` property defaults to a singular version of the class name. While plural names are technically supported, singular is the opinionated standard across all framework docs, generators, and migrations.

- **$table**: The database table name.
- **$primaryKey**: The column representing the primary key. Defaults to `id`.
- **$fillable**: An array of attributes that are mass-assignable.
- **$guarded**: Attributes that should be protected from mass-assignment.
- **$casts**: Defines how attributes should be cast when retrieved (e.g., `json`, `datetime`, `int`, `enum`).
- **$timestamps**: Whether to automatically manage `created_at` and `updated_at`. Defaults to `true`.
- **$hidden**: Attributes that should be hidden from serialization.
- **$visible**: Attributes that should be included in serialization (whitelist).
- **$appends**: Accessors to include in the model's array/JSON representation.
- **$rules**: Validation rules for the model.

### Generating Models

```bash
php dock model:create Post Post
```

### Generating Models from Migrations

Use the `--migration` flag to automatically generate a model with inferred properties from an existing migration. Both the class name and the full migration filename are accepted:

```bash
# Using class name
php dock model:create Post Post --migration=CreatePostTable

# Using full migration filename
php dock model:create Post Post -m 2025_10_26_191939_create_post_table
```

This parses the migration file and generates a model with `$fillable`, `$casts`, `$hidden`, relationship methods, query scopes, and PHPDoc `@property` annotations. See the [CLI docs](cli.md#smart-model-generation) for details.

## Retrieving Models

### all

```php
static all(array $columns = ['*']): ModelCollection
```

Retrieves all records from the database table.

- **Use Case**: Listing all active categories for a navigation menu.

### find

```php
static find(int|string $id): ?static
```

Retrieves a single record by its primary key.

- **Example**: `$user = User::find(1);`.

### findMany

```php
static findMany(array $ids, array $columns = ['*']): ModelCollection
```

Retrieves multiple records by their primary keys.

### query

```php
static query(): Builder
```

Initiates a fluent query builder instance for complex filtering.

- **Example**: `User::query()->where('active', 1)->orderBy('name')->get();`.

### firstOrCreate / updateOrCreate

```php
static firstOrCreate(array $attributes, array $values = []): static
static updateOrCreate(array $attributes, array $values = []): static
```

Find or create a model based on matched attributes.

## Creating & Saving

### save

```php
save(): bool
```

Persists the model instance to the database (performs an `INSERT` if new, or `UPDATE` if existing).

- **Use Case**: Manually building a model instance before saving.

### create

```php
static create(array $attributes): static
```

Mass-assigns and saves a new model in a single step.

- **Note**: Attributes must be listed in the `$fillable` array.
- **Example**: `User::create(['name' => 'Ben', 'email' => '...'])`.

### increment / decrement

```php
increment(string $column, float|int $amount = 1, array $extra = []): int
decrement(string $column, float|int $amount = 1, array $extra = []): int
```

Increment or decrement the value of a column.

## Updating

```php
$tweet = Tweet::find(1);
$tweet->message = 'New Message';
$tweet->save();

// Or use update method
$tweet->update(['message' => 'New Message']);
```

## Deleting

```php
$tweet = Tweet::find(1);
$tweet->delete();

// Soft delete (if enabled)
// Ensure `protected bool $softDeletes = true;` is set in your model
$tweet->delete(); // Marks as deleted

// Force delete (permanent)
$tweet->forceDelete();

// Restore soft deleted
$tweet->restore();
```

### Enabling Soft Deletes

To enable soft deletes, add the property to your model:

```php
class Tweet extends BaseModel
{
    protected bool $softDeletes = true;
}
```

## Accessors & Mutators

### Accessors

Accessors allow you to format model attributes when they are retrieved. To define an accessor, create a `get{Name}Attribute` method:

```php
class User extends BaseModel
{
    public function getFullNameAttribute(): string
    {
        return "{$this->first_name} {$this->last_name}";
    }
}

// Usage
echo $user->full_name;
```

### Mutators

Mutators allow you to format model attributes when they are set. To define a mutator, create a `set{Name}Attribute` method:

```php
class User extends BaseModel
{
    public function setPasswordAttribute(string $value): void
    {
        $this->attributes['password'] = password_hash($value, PASSWORD_BCRYPT);
    }
}

// Usage
$user->password = 'secret';
```

## Query Scopes

### Local Scopes

Local scopes allow you to define common sets of constraints that you may easily re-use. To define a scope, prefix a model method with `scope`:

```php
class User extends BaseModel
{
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('active', true);
    }

    public function scopeOfType(Builder $query, string $type): Builder
    {
        return $query->where('type', $type);
    }
}

// Usage
$users = User::active()->get();
$admins = User::active()->ofType('admin')->get();
```

### Global Scopes

Global scopes are applied to all queries for a model.

```php
class User extends BaseModel
{
    protected static function boot(): void
    {
        parent::boot();

        static::addGlobalScope('active', function(Builder $builder) {
            $builder->where('active', true);
        });
    }
}

// Remove global scope
User::withoutGlobalScope('active')->get();
```

## Model Lifecycle

Models fire several events throughout their lifecycle that you can hook into using either **Method Callbacks** (preferred for model-specific logic) or **Static Event Listeners** (preferred for Traits and global logic).

### Method Hooks (Callbacks)

Similar to Ruby on Rails, you can simply define a method on your model to react to a lifecycle stage. These methods are automatically discovered by the framework.

| Hook | When it fires | Purpose |
| :--- | :--- | :--- |
| `onRetrieved` | After record is loaded | Decryption, formatting data. |
| `onSaving` | Before Insert OR Update | Validation, normalization, setting defaults. |
| `onSaved` | After Insert OR Update | External API sync, logging. |
| `onCreating` | Before new record insert | Generating RefIDs, complex defaults. |
| `onCreated` | After new record insert | Sending welcome emails, initial setup. |
| `onUpdating` | Before existing record update | Change detection. |
| `onUpdated` | After existing record update | Auditing, cache clearing. |
| `onDeleting` | Before record is removed | Asset cleanup, relationship checks. |
| `onDeleted` | After record is removed | Post-deletion cleanup. |

#### Example: Normalization & Halting

If a hook returns `false`, the operation (save/delete) will be halted.

```php
class User extends BaseModel
{
    /**
     * Halts the save process if the user is too young.
     */
    protected function onSaving(): ?bool
    {
        $this->name = trim($this->name);
        
        if ($this->age < 18) {
            return false; // Prevents the save()
        }
        
        return true;
    }
}
```

#### Example: Post-Creation Logic

Useful for actions that require the model to have an `id` (primary key) assigned first.

```php
class Order extends BaseModel
{
    protected function onCreated(): void
    {
        // Now that order is saved and has an ID...
        notify('orders')->send(new NewOrderNotification($this));
        
        // Push a job to handle heavy processing
        queue(ProcessOrderJob::class, ['order_id' => $this->id]);
    }
}
```

#### Example: Resource Cleanup

Use `onDeleting` to clean up physical files or related resources before the database record disappears.

```php
class Media extends BaseModel
{
    protected function onDeleting(): void
    {
        // Delete the actual file from disk before removing the DB record
        FileSystem::delete($this->path);
    }
}
```

### Static Event Listeners

Static listeners are useful when you want to define behavior in a **Trait** or from an external Service Provider.

```php
class User extends BaseModel
{
    protected static function boot(): void
    {
        parent::boot();

        static::creating(function(User $user) {
            $user->ref_id = Str::refid();
        });
    }
}
```

### Best Practices for Lifecycle Hooks

- **Use Method Hooks for Internal Logic**: If the logic belongs strictly inside that model (e.g., calculating a total, trimming names), use `onSaving()`. It is faster to find and read.
- **Use Static Listeners for Traits**: If you are writing a reusable Trait (like `HasRefId`), use the `bootTraitName` approach with `static::creating`. This prevents method name collisions.
- **Avoid Heavy Logic**: Don't perform long-running tasks (like resizing images) directly in these hooks. Instead, use the `onCreated` hook to dispatch a **Queue Job**.
- **Return Types**: Always include the `?bool` return type for hooks that can halt execution (`onSaving`, `onCreating`, `onUpdating`, `onDeleting`).

## Attribute Casting

The `$casts` property provides a convenient way to convert attributes to common data types.

Supported types: `int`, `float`, `string`, `bool`, `date`, `datetime`, `json`, `array`, `enum`.

```php
class User extends BaseModel
{
    protected array $casts = [
        'is_active' => 'bool',
        'meta' => 'json',
        'joined_at' => 'datetime',
        'status' => UserStatus::class, // Enum casting
    ];
}
```

## Relationships

### One to One

#### `hasOne`

A child model is linked to a parent through a foreign key.

- **Use Case**: A `User` having a single `Profile` or a `Company` having one `Settings` record.
- **Example**:

```php
public function profile(): HasOne {
    return $this->hasOne(Profile::class);
}
```

### One to Many

#### `hasMany`

Define a one-to-many relationship. For example, a User has many Tweets:

```php
class User extends BaseModel
{
    public function tweets(): HasMany
    {
        return $this->hasMany(Tweet::class);
    }
}

// Usage
$user = User::find(1);
$tweets = $user->tweets;
```

### Inverse Relationship

#### `belongsTo`

Define the inverse of a one-to-one or one-to-many relationship:

```php
class Tweet extends BaseModel
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}

// Usage
$tweet = Tweet::find(1);
$user = $tweet->user;
```

### Many to Many

#### `belongsToMany`

Define a many-to-many relationship using a pivot table. For example, Users and Roles:

```php
class User extends BaseModel
{
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
        // Assumes pivot table: role_user (alphabetical)
        // With columns: role_id, user_id
    }
}

// Custom pivot table and keys
public function roles(): BelongsToMany
{
    return $this->belongsToMany(
        Role::class,
        'user_roles',        // pivot table
        'user_id',           // foreign key on pivot
        'role_id'            // related key on pivot
    );
}

// Usage
$user = User::find(1);
$roles = $user->roles;

// Access pivot data
foreach ($user->roles as $role) {
    echo $role->pivot['created_at'];
}
```

### Has Many Through

`hasManyThrough`

Define a distant relationship through an intermediate model. For example, a Country has many Posts through Users:

```php
class Country extends BaseModel
{
    public function posts(): HasManyThrough
    {
        return $this->hasManyThrough(
            Post::class,     // Final model
            User::class      // Intermediate model
        );
    }
}

// Usage
$country = Country::find(1);
$posts = $country->posts;
```

### Polymorphic Relationships

Polymorphic relationships allow a model to belong to more than one other single model on a single association.

#### morphOne

Imagine you have a `users` table and a `posts` table. Both users and posts need to have a "featured image". Instead of creating two separate tables like `user_images` and `post_images`, you can create a single `images` table that stores images for both models. This is a perfect use case for a **One to One Polymorphic Relationship**.

In this example, `Post` and `User` share a polymorphic relation to `Image`.

```php
class Post extends BaseModel
{
    public function image()
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

class User extends BaseModel
{
    public function image()
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}
```

#### morphTo

To complete the relationship, the `Image` model needs to know how to determine which model it belongs to.

```php
class Image extends BaseModel
{
    public function imageable()
    {
        return $this->morphTo();
    }
}
```

#### morphMany

Consider a scenario where users can leave comments on both `Videos` and `Posts`. Instead of creating `video_comments` and `post_comments` tables, you can use a single `comments` table to handle comments for both types of content. This is achieved using a **One to Many Polymorphic Relationship**.

```php
class Post extends BaseModel
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends BaseModel
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

#### Retrieving Polymorphic Relations

```php
$post = Post::find(1);
$image = $post->image;

$image = Image::find(1);
$imageable = $image->imageable; // Returns Post or User instance
```

## Eager Loading

Prevent N+1 query problems by eager loading relationships:

```php
// Eager load one relationship
$users = User::with('tweets')->get();

// Eager load multiple relationships
$users = User::with(['tweets', 'profile'])->get();

// Nested eager loading
$users = User::with('tweets.comments')->get();

// Eager load with constraints
$users = User::with(['tweets' => function($query) {
    $query->where('published', true);
}])->get();

// Eager load specific columns
$users = User::with('tweets:id,message,user_id')->get();
```

## Lazy Eager Loading

Load relationships after retrieving the model:

```php
$user = User::find(1);

// Load a relationship
$user->load('tweets');

// Load multiple relationships
$user->load(['tweets', 'profile']);

// Load only if not already loaded
$user->loadMissing('tweets');
```

## Counting Related Models

```php
// Load relationship counts
$users = User::withCount('tweets')->get();

foreach ($users as $user) {
    echo $user->tweets_count;
}

// Count with constraints
$users = User::withCount(['tweets' => function($query) {
    $query->where('published', true);
}])->get();

// Load counts on existing model
$user->loadCount('tweets');
```

## Querying Relationships

```php
// Query relationship existence
$users = User::whereHas('tweets')->get();

// Query with constraints
$users = User::whereHas('tweets', function($query) {
    $query->where('published', true);
})->get();
```

## Relationship Methods

When defining relationships, you can customize foreign keys:

```php
// hasOne / hasMany
$this->hasOne(Profile::class, 'user_id', 'id');
//                            foreign key  local key

// belongsTo
$this->belongsTo(User::class, 'user_id', 'id');
//                            foreign key  owner key

// belongsToMany
$this->belongsToMany(
    Role::class,
    'role_user',      // pivot table
    'user_id',        // foreign pivot key
    'role_id',        // related pivot key
    'id',             // parent key
    'id'              // related key
);

// hasManyThrough
$this->hasManyThrough(
    Post::class,      // related model
    User::class,      // through model
    'country_id',     // foreign key on through model
    'user_id',        // foreign key on related model
    'id',             // local key on parent
    'id'              // local key on through model
);

// morphTo
$this->morphTo(
    'imageable',      // relationship name (optional)
    'imageable_type', // type column (optional)
    'imageable_id'    // id column (optional)
);

// morphOne
$this->morphOne(
    Image::class,     // related model
    'imageable',      // relationship name
    'imageable_type', // type column (optional)
    'imageable_id',   // id column (optional)
    'id'              // local key (optional)
);

// morphMany
$this->morphMany(
    Comment::class,   // related model
    'commentable',    // relationship name
    'commentable_type', // type column (optional)
    'commentable_id',   // id column (optional)
    'id'                // local key (optional)
);
```

## Serialization

Anchor provides methods for converting your models to arrays or JSON.

```php
$user = User::find(1);

// Convert to array
$user->toArray();

// Convert to JSON
$user->toJson();

// Hide attributes
protected array $hidden = ['password'];

// Whitelist attributes
protected array $visible = ['first_name', 'last_name'];

// Append accessors
protected array $appends = ['full_name'];
```

## Model State and Persistence

### State Helpers

```php
$user = User::find(1);

// Check if attributes have changed
if ($user->isDirty()) {
    // ...
}

// Check if a specific attribute has changed
if ($user->isDirty('email')) {
    // ...
}

// Get the original attribute values
$user->getOriginal();

// Reload the model from the database
$user->refresh();

// Get a fresh instance of the model
$freshUser = $user->fresh();
```

### Persistence Utilities

```php
// Find or create
$user = User::firstOrCreate(['email' => '...'], ['name' => 'Ben']);

// Update or create
$user = User::updateOrCreate(['email' => '...'], ['name' => 'New Name']);
```

## Validation

Models can define `$rules` for automated validation during `save()`.

```php
class User extends BaseModel
{
    protected array $rules = [
        'email' => 'required|email|unique:user',
        'age' => 'integer|min:18',
    ];
}
```

## Lazy Collections

For memory-efficient processing of large datasets:

```php
foreach (User::lazy() as $user) {
    // Process one user at a time
}
```

## Auto Eager Loading

Automatically load relationships for all queries:

```php
class User extends BaseModel
{
    protected static bool $autoEagerLoadEnabled = true;
}
```
