# Database Migrations

Migrations are like version control for your database, allowing you to define and share the application's database schema definition.

## Creating Migrations

Use the CLI to generate a migration:

```bash
php dock migration:create CreateUsersTable
```

This creates a timestamped file in `App/storage/database/migrations/`.

## Migration File Location

Migrations are stored in:

```
App/storage/database/migrations/
```

File naming convention:

```
YYYY_MM_DD_HHMMSS_MigrationName.php
```

## Migration Structure

Migrations extend `Database\Migration\BaseMigration` and contain `up()` and `down()` methods:

```php
<?php
namespace Database\Migrations;

use Database\Migration\BaseMigration;
use Database\Schema\SchemaBuilder;

class CreateUserTable extends BaseMigration
{
    public function up(): void
    {
        $this->schema()->create('user', function (SchemaBuilder $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        $this->schema()->dropIfExists('user');
    }
}
```

## Available Column Types

### Numeric Types

```php
$table->id();                          // Auto-incrementing primary key
$table->bigInteger('votes');           // BIGINT
$table->integer('votes');              // INTEGER
$table->mediumInteger('votes');        // MEDIUMINT
$table->smallInteger('votes');         // SMALLINT
$table->tinyInteger('votes');          // TINYINT
$table->decimal('amount', 8, 2);       // DECIMAL with precision
$table->double('amount', 8, 2);        // DOUBLE
$table->float('amount', 8, 2);         // FLOAT
$table->unsignedBigInteger('user_id'); // BIGINT UNSIGNED
$table->unsignedInteger('votes');      // INT UNSIGNED
$table->unsignedSmallInteger('votes'); // SMALLINT UNSIGNED
$table->unsignedTinyInteger('votes');  // TINYINT UNSIGNED
```

### String Types

```php
$table->string('name');                // VARCHAR(255)
$table->string('name', 100);           // VARCHAR(100)
$table->text('description');           // TEXT
$table->tinyText('summary');           // TINYTEXT
$table->mediumText('content');         // MEDIUMTEXT
$table->longText('description');       // LONGTEXT
$table->char('code', 4);               // CHAR
$table->uuid('uuid');                  // UUID
$table->ipAddress('ip');               // IP Address (VARCHAR(45))
$table->binary('data');                // BLOB/BINARY
```

### Date & Time

```php
$table->date('created_date');          // DATE
$table->datetime('created_at');        // DATETIME
$table->timestamp('created_at');       // TIMESTAMP
$table->dateTimestamps();              // created_at & updated_at (DATETIME) - Opinionated preference
$table->timestamps();                  // created_at & updated_at (TIMESTAMP)
$table->softDeletes();                 // deleted_at (TIMESTAMP)
$table->softDeletesTz();               // deleted_at (TIMESTAMP with TZ support)
$table->year('birth_year');            // YEAR
```

### Boolean

```php
$table->boolean('confirmed');          // BOOLEAN/TINYINT(1)
```

### JSON

```php
$table->json('options');               // JSON
```

## Column Modifiers

```php
$table->string('email')->nullable();           // Allow NULL
$table->string('email')->unique();             // Add unique index
$table->string('name')->default('Guest');      // Default value
$table->integer('votes')->unsigned();          // Unsigned
$table->string('email')->index();              // Add index
$table->text('bio')->comment('User bio');      // Add comment
$table->string('address')->after('email');     // Position (MySQL only)
$table->string('status')->columnComment('User status'); // Explicit comment
$table->timestamp('updated_at')->onUpdateRaw('CURRENT_TIMESTAMP'); // Raw ON UPDATE
```

## Foreign Keys

```php
$table->integer('user_id')->unsigned();
$table->foreign('user_id')
    ->references('id')
    ->on('user')
    ->onDelete('cascade');

// Or manually
$table->integer('user_id')->unsigned();
$table->foreign('user_id')
    ->references('id')
    ->on('user')
    ->onDelete('cascade')
    ->onUpdate('cascade');
```

## Indexes

```php
$table->index('email');                        // Single column
$table->index(['first_name', 'last_name']);    // Composite
$table->unique('email');                       // Unique index
$table->primary('id');                         // Primary key
$table->fulltext('description');               // Fulltext index (MySQL only)
```

## Modifying Tables

### Adding Columns

```php
public function up(): void
{
    $this->schema()->table('user', function (SchemaBuilder $table) {
        $table->string('phone')->nullable();
        $table->string('address')->after('email');
        $table->raw('ALTER TABLE user ADD COLUMN custom_data JSON'); // Raw DDL
    });
}
```

### Dropping Columns

```php
public function up(): void
{
    $this->schema()->table('user', function (SchemaBuilder $table) {
        $table->dropColumn('phone');
        $table->dropColumn(['phone', 'address']); // Multiple
        $table->dropColumnIfExists('social_id');   // Safe drop
    });
}
```

### Renaming Columns

```php
public function up(): void
{
    $this->schema()->table('user', function (SchemaBuilder $table) {
        $table->renameColumn('name', 'full_name', 'VARCHAR(255)');
    });
}
```

### Table Options

**MySQL/MariaDB Only**

```php
$this->schema()->create('logs', function (SchemaBuilder $table) {
    $table->id();
    $table->text('message');

    $table->engine('InnoDB');
    $table->charset('utf8mb4');
    $table->collation('utf8mb4_unicode_ci');
    $table->comment('Application logs');
});
```

## Driver-Specific Logic

Some database drivers, like SQLite, have limitations on certain schema operations (e.g., adding foreign keys to existing tables). You can use driver-specific methods to handle these cases gracefully.

- `whenDriverIs(string|array $drivers, callable $callback)`

Execute schema modifications only if the current driver matches.

```php
Schema::whenDriverIs('mysql', function () {
    // MySQL specific logic
});

// Within a table callback
$table->whenDriverIs('pgsql', function ($table) {
    // Postgres specific logic
});
```

`whenDriverIsNot(string|array $drivers, callable $callback)`

Execute schema modifications only if the current driver does NOT match. This is particularly useful for skipping unsupported SQLite operations.

```php
Schema::whenDriverIsNot('sqlite', function () {
    Schema::table('users', function (SchemaBuilder $table) {
        $table->foreign('role_id')->references('id')->on('roles');
    });
});
```

## Running Migrations

### Run All Pending Migrations

```bash
php dock migration:run
```

### Check Migration Status

```bash
php dock migration:status
```

### List All Migrations

```bash
php dock migration:list
```

### Run Specific Migrations

Run one or more specific migration files:

```bash
# Run single migration
php dock migration:run --file=2024_01_15_143022_create_users_table

# Run multiple migrations (comma-separated)
php dock migration:run --file=2024_01_15_143022_create_users_table,2024_01_16_120000_create_posts_table
```

> Use `--file` when you want to run only specific migrations without running all pending migrations. This is useful for package installations or selective schema updates.

## Rolling Back Migrations

### Rollback Last Batch

```bash
php dock migration:rollback
```

### Rollback Specific Migrations

Rollback one or more specific migration files:

```bash
# Rollback single migration
php dock migration:rollback --file=2024_01_15_143022_create_users_table

# Rollback multiple migrations (comma-separated)
php dock migration:rollback --file=2024_01_15_143022_create_users_table,2024_01_16_120000_create_posts_table
```

> When using `--file` with rollback, ensure you rollback migrations in the correct order (reverse of creation) to avoid foreign key constraint issues.

### Rollback All Migrations

```bash
php dock migration:reset
```

### Refresh Database

**rollback and re-run**

```bash
php dock migration:refresh
```

## Programmatic Migration Execution

For advanced use cases, you can run specific migrations programmatically:

```php
use Database\Migration\Migrator;
use Database\ConnectionInterface;
use Database\Helpers\DatabaseOperationConfig;

$connection = resolve(ConnectionInterface::class);
$config = resolve(DatabaseOperationConfig::class);
$migrator = new Migrator($connection, $config->getMigrationsPath());

// Run specific migration files
$results = $migrator->runFiles([
    '2024_01_15_143022_create_users_table',
    '2024_01_16_120000_create_posts_table'
]);

// Get pending migrations
$pending = $migrator->getPendingMigrations();

// Run all pending
$results = $migrator->run();
```

## Migration Locking

Prevent concurrent migrations:

```bash
# Lock migrations
php dock migration:lock

# Unlock migrations
php dock migration:unlock
```

## Table Definitions

### User Table

```php
class CreateUserTable extends BaseMigration
{
    public function up(): void
    {
        $this->schema()->create('user', function (SchemaBuilder $table) {
            $table->id();
            $table->string('refid', 32)->unique();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->string('phone')->nullable();
            $table->enum('status', ['active', 'inactive', 'suspended'])->default('inactive');
            $table->timestamp('email_verified_at')->nullable();
            $table->timestamp('password_updated_at')->nullable();
            $table->dateTimestamps();
            $table->softDeletes();

            $table->index('email');
            $table->index('status');
        });
    }

    public function down(): void
    {
        $this->schema()->dropIfExists('user');
    }
}
```

### Post Table with Foreign Key

```php
class CreatePostTable extends BaseMigration
{
    public function up(): void
    {
        $this->schema()->create('post', function (SchemaBuilder $table) {
            $table->id();
            $table->integer('user_id')->unsigned();
            $table->foreign('user_id')
                ->references('id')
                ->on('user')
                ->onDelete('cascade');
            $table->string('title');
            $table->text('content');
            $table->string('slug')->unique();
            $table->boolean('published')->default(false);
            $table->timestamp('published_at')->nullable();
            $table->dateTimestamps();

            $table->index(['user_id', 'published']);
            $table->index('slug');
        });
    }

    public function down(): void
    {
        $this->schema()->dropIfExists('post');
    }
}
```

### Pivot Table

```php
class CreateRoleUserTable extends BaseMigration
{
    public function up(): void
    {
        $this->schema()->create('role_user', function (SchemaBuilder $table) {
            $table->id();
            $table->integer('role_id')->unsigned();
            $table->foreign('role_id')->references('id')->on('role')->onDelete('cascade');
            $table->integer('user_id')->unsigned();
            $table->foreign('user_id')->references('id')->on('user')->onDelete('cascade');
            $table->dateTimestamps();

            $table->unique(['role_id', 'user_id']);
        });
    }

    public function down(): void
    {
        $this->schema()->dropIfExists('role_user');
    }
}
```

### Adding Column to Existing Table

```php
class AddPhoneToUserTable extends BaseMigration
{
    public function up(): void
    {
        $this->schema()->table('user', function (SchemaBuilder $table) {
            $table->string('phone', 20)->nullable()->after('email');
            $table->index('phone');
        });
    }

    public function down(): void
    {
        $this->schema()->table('user', function (SchemaBuilder $table) {
            $table->dropIndex(['phone']);
            $table->dropColumn('phone');
        });
    }
}
```

## Best Practices

1. **Use singular table names**: Table names follow the **singular** convention (e.g., `user`, `post`, `order`). This matches the model class name and is the framework's opinionated preference.
2. **Always define down()**: Make migrations reversible
3. **Use foreign keys**: Maintain referential integrity
4. **Add indexes**: For frequently queried columns
5. **Use timestamps**: Track when records are created/updated
6. **Test rollbacks**: Ensure down() works correctly
7. **One change per migration**: Keep migrations focused
8. **Use descriptive names**: Make migration purpose clear
9. **Order matters**: Create parent tables before children

## Schema Methods

```php
// Create table
$this->schema()->create('user', function (SchemaBuilder $table) {
    // ...
});

// Modify table
$this->schema()->table('user', function (SchemaBuilder $table) {
    // ...
});

// Drop table
$this->schema()->drop('user');

// Drop if exists
$this->schema()->dropIfExists('user');

// Rename table
$this->schema()->rename('user', 'app_user');

// Check if index exists
if ($this->schema()->hasIndex('user', 'user_email_index')) {
    // ...
}

// Check if unique key exists
if ($this->schema()->hasUnique('user', 'user_email_unique')) {
    // ...
}

// Check if column exists
if ($this->schema()->hasColumn('user', 'email')) {
    // ...
}

// Check if foreign key exists
if ($this->schema()->hasForeignKey('user', 'user_role_id_foreign')) {
    // ...
}

// Check if table exists
if ($this->schema()->hasTable('user')) {
    // ...
}

// Truncate table
$this->schema()->truncate('user');

// Drop methods
$this->schema()->dropIndex('user', 'index_name');
$this->schema()->dropUnique('user', 'unique_name');
$this->schema()->dropForeign('user', 'foreign_key');
$this->schema()->dropPrimary('user');

// Safe Drop Table
$this->schema()->dropIfExists('user');
```

## SchemaBuilder Methods

These methods are available on the `$table` (SchemaBuilder) instance within `create()` or `table()` callbacks.

### Modifying Columns

#### `change()`

Modify an existing column's type or attributes.

```php
$table->string('email', 100)->nullable()->change();
```

### Unique Keys

#### `uniqueKey()`

Add a unique key constraint.

```php
$table->uniqueKey(['email', 'status'], 'user_unique_email_status');
```

### Conditional Operations

The Schema class provides methods to conditionally execute operations, making migrations idempotent and safe to run multiple times.

#### Environment-Specific Operations

- `whenEnvironment(string|array $environments, callable $callback)`
- `whenNotEnvironment(string|array $environments, callable $callback)`

```php
// Only create this index in production
$table->whenEnvironment('prod', function ($table) {
    $table->index('search_vector');
});

// Skip this heavy index in local dev
$table->whenNotEnvironment('dev', function ($table) {
    $table->index(['log_level', 'created_at']);
});
```

#### Index Operations

| Method                                         | Description            |
| :--------------------------------------------- | :--------------------- |
| `whenTableHasIndex($table, $idx, $cb)`         | Run if index exists.   |
| `whenTableDoesntHaveIndex($table, $idx, $cb)`  | Run if index missing.  |
| `whenTableHasColumn($table, $col, $cb)`        | Run if column exists.  |
| `whenTableDoesntHaveColumn($table, $col, $cb)` | Run if column missing. |

#### Fluent Idempotent Operations

These methods are available directly on the `SchemaBuilder` (inside `table()` callbacks) and return the builder instance for chaining.

| Method                                  | Description            |
| :-------------------------------------- | :--------------------- |
| `indexIfNotExist($cols, $name = null)`  | Add index if missing.  |
| `uniqueIfNotExist($cols, $name = null)` | Add unique if missing. |
| `foreignIfNotExist($col)`               | Add FK if missing.     |
| `dropIndexIfExists($name)`              | Drop index if exists.  |
| `dropUniqueIfExists($name)`             | Drop unique if exists. |
| `dropForeignIfExists($name)`            | Drop FK if exists.     |
| `dropColumnIfExists($cols)`             | Drop column if exists. |

#### Foreign Key Operations

| Method                               | Description           |
| :----------------------------------- | :-------------------- |
| `dropForeignIfExists($table, $name)` | Drop FK from table.   |
| `foreignIfNotExist($column)`         | Fluent FK on builder. |

```php
$this->schema()->table('user', function (SchemaBuilder $table) {
    // Only drops if it exists
    $table->dropForeignIfExists('user_role_id_foreign');

    // Only adds if it doesn't exist
    $table->foreignIfNotExist('role_id')
        ->references('id')
        ->on('role');
});
```

### Advanced Migration Patterns:

#### Idempotent Migrations

Make migrations safe to run multiple times:

```php
class AddProductIndexes extends BaseMigration
{
    public function up(): void
    {
        // Safe to run multiple times - won't error if index exists
        $this->schema()->whenTableDoesntHaveIndex('product', 'product_sku_index', function($table) {
            $table->index('sku', 'product_sku_index');
        });

        $this->schema()->whenTableDoesntHaveIndex('product', 'product_category_status_index', function($table) {
            $table->index(['category_id', 'status'], 'product_category_status_index');
        });
    }

    public function down(): void
    {
        // Safe rollback - won't error if index doesn't exist
        $this->schema()->whenTableHasIndex('product', 'product_sku_index', function($table) {
            $table->dropIndex('product_sku_index');
        });

        $this->schema()->whenTableHasIndex('product', 'product_category_status_index', function($table) {
            $table->dropIndex('product_category_status_index');
        });
    }
}
```

#### Environment-Specific Schemas

Handle differences between dev/staging/prod:

```php
public function up(): void
{
    $this->schema()->table('order', function ($table) {
        // Add performance index only in production
        $table->whenEnvironment('prod', function($table) {
            $table->index('search_vector');
        });

        // Add debugging columns only in development
        $table->whenEnvironment('dev', function($table) {
            $table->string('debug_info')->nullable();
        });
    });
}
```

#### Package Migrations

Safe installation/uninstallation:

```php
// Package installation migration
public function up(): void
{
    $this->schema()->whenTableDoesntHaveIndex('user', 'user_tenant_id_index', function($table) {
        $table->index('tenant_id', 'user_tenant_id_index');
    });
}

// Package uninstallation
public function down(): void
{
    $this->schema()->whenTableHasIndex('user', 'user_tenant_id_index', function($table) {
        $table->dropIndex('user_tenant_id_index');
    });
}
```

#### Incremental Updates

Add indexes only where needed:

```php
public function up(): void
{
    // Check multiple tables and add indexes as needed
    $tables = ['user', 'post', 'comment'];

    foreach ($tables as $tableName) {
        $indexName = "{$tableName}_created_at_index";
        $this->schema()->whenTableDoesntHaveIndex($tableName, $indexName, function($table) use ($indexName) {
            $table->index('created_at', $indexName);
        });
    }
}
```
