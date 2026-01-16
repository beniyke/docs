# Database

Anchor provides a powerful database layer with support for MySQL, MariaDB, PostgreSQL, and SQLite. You can interact with the database using the `DB` facade, the Query Builder (preferred), or raw SQL.

## Configuration

Database settings are in `App/Config/database.php`:

```php
return [
    'driver' => env('DB_CONNECTION', 'sqlite'),

    'connections' => [
        'mysql' => [
            'host' => env('DB_HOST', 'localhost'),
            'name' => env('DB_DATABASE', 'anchor'),
            'user' => env('DB_USERNAME', 'root'),
            'password' => env('DB_PASSWORD', ''),
            'driver' => 'mysql',
            'persistent' => true,
        ],

        'sqlite' => [
            'path' => 'App/storage/database',
            'file' => env('DB_DATABASE', 'anchor.sqlite'),
            'persistent' => true,
        ],
    ],
];
```

> **SQLite Compatibility**: The framework provides enhanced SQLite support, including the ability to run multiple `ADD COLUMN` operations within a single `ALTER TABLE` schema command.

## Getting Started

The `DB` facade provides a static interface to the database.

```php
use Database\DB;

// Query Builder (Preferred)
$users = DB::table('users')->where('active', '=', 1)->get();

// Table Select Shortcut
$results = DB::select('users'); // Equivalent to DB::table('users')->get()

// Transactions
DB::transaction(function() {
    DB::table('users')->insert(['name' => 'John']);
});
```

### Query Builder Overview

Anchor's Query Builder provides a fluent API for complex queries. For a complete list of all available methods, see the [Query Builder Reference](query-builder.md).

#### Advanced Where Clauses

| Method                                 | Description                        |
| :------------------------------------- | :--------------------------------- |
| `whereIn`, `whereNotIn`                | Filter by a list of values.        |
| `whereBetween`, `whereNotBetween`      | Filter by a range.                 |
| `whereNull`, `whereNotNull`            | Filter by NULL/NOT NULL.           |
| `whereExists`, `whereNotExists`        | Existence checks using subqueries. |
| `whereDate`, `whereMonth`, `whereYear` | Date-part comparisons.             |
| `whereJsonContains`, `whereJsonLength` | Query JSON columns.                |

#### Conditional Queries

Use the `when()` method to apply clauses conditionally:

```php
$role = request('role');

$users = DB::table('users')
    ->when($role, function ($query, $role) {
        return $query->where('role_id', $role);
    })
    ->get();
```

#### Inserts, Updates & Deletes

```php
// Insert and get ID
$id = DB::table('users')->insertGetId(['name' => 'John']);

// Insert or ignore duplicates
DB::table('users')->insertOrIgnore(['email' => 'duplicate@example.com']);

// Update rows
DB::table('users')->where('id', 1)->update(['active' => 1]);

// Delete rows
DB::table('users')->where('active', 0)->delete();
```

## Transactions

**Using DB::transaction()**

```php
use Database\DB;

DB::transaction(function() {
    DB::table('users')->insert(['name' => 'John']);
    DB::table('posts')->insert(['user_id' => 1, 'title' => 'Hello']);

    // If exception is thrown, transaction is rolled back
    // If successful, transaction is committed
});
```

**Manual Transaction Control**

```php
$connection = DB::connection();

$connection->beginTransaction();

try {
    // Operations...
    DB::table('users')->insert(['name' => 'Jane']);

    $connection->commit();
} catch (\Exception $e) {
    $connection->rollBack();
    throw $e;
}
```

**Transaction Callbacks**

```php
DB::afterCommit(function() {
    // Runs after successful commit
    logger('database.log')->info('Transaction committed');
});

DB::afterRollback(function() {
    // Runs after rollback
    logger('database.log')->error('Transaction rolled back');
});
```

## Raw SQL Queries

While the Query Builder is preferred, you can run raw SQL for complex or specific use cases.

**Select Queries**

```php
// Get all results
$users = DB::connection()->select('SELECT * FROM users WHERE active = ?', [1]);

// Get single result
$user = DB::connection()->selectOne('SELECT * FROM users WHERE id = ?', [1]);
```

**Insert, Update, Delete**

```php
// Insert
DB::connection()->statement('INSERT INTO users (name) VALUES (?)', ['John']);

// Update
$affected = DB::connection()->update('UPDATE users SET name = ? WHERE id = ?', ['Jane', 1]);

// Delete
$deleted = DB::connection()->delete('DELETE FROM users WHERE id = ?', [1]);
```

**Generic Statement**

```php
DB::connection()->statement('CREATE INDEX idx_email ON users(email)');
```

**Raw Expressions in Builder**

```php
// In select
$users = DB::table('users')
    ->select([DB::raw('COUNT(*) as total')])
    ->get();

// In where
$users = DB::table('users')
    ->whereRaw('YEAR(created_at) = ?', [2024])
    ->get();
```

## Database Schema Operations

Methods to inspect and modify database schema.

**Table Operations**

```php
$tables = DB::connection()->getTables();

if (DB::connection()->tableExists('users')) {
    // Table exists
}

DB::connection()->truncateTable('users');
```

### Column Operations

```php
if (DB::connection()->columnExists('users', 'email')) {
    // Column exists
}
```

## Advanced Features

### Query Logging

```php
use Database\Connection;

// Enable implicit logging by calling:
$log = Connection::getQueryLog();

foreach ($log as $query) {
    // Process log
}

Connection::clearQueryLog();
```

### Slow Query Detection

Monitor slow queries globally:

```php
DB::whenQueryingForLongerThan(0.5, function($query) {
    logger('database.log')->warning('Slow query detected', [
        'sql' => $query['sql'],
        'time_ms' => $query['time_ms']
    ]);
});
```

### Prepared Statement Caching

The framework automatically caches prepared statements for performance.

```php
$stats = DB::connection()->getCacheStats();
// ['hits' => 10, 'misses' => 1, ...]

DB::connection()->setMaxCacheSize(200);
DB::connection()->clearStatementCache();
```

## Connection Management

### Multiple Connections

Anchor supports multiple active database connections. By default, the `DB` facade uses the connection specified by your `driver` config.

#### Configuration

Add named connections to the `connections` array in `App/Config/database.php`:

```php
'connections' => [
    'default' => [ ... ],
    'reporting' => [
        'driver' => 'mysql',
        'host' => 'reports.example.com',
        // ...
    ],
    'audit' => [
        'driver' => 'sqlite',
        'database' => 'audit.sqlite',
        // ...
    ],
],
```

#### Usage

Use `DB::connection($name)` to switch to a specific connection:

```php
// Use the reporting database
$stats = DB::connection('reporting')->table('metrics')->count();

// Use the audit database
DB::connection('audit')->table('logs')->insert([
    'action' => 'login',
    'timestamp' => now()
]);
```

**Lazy Loading**: If you request a connection that hasn't been established yet, the framework will automatically establish it using the settings in your configuration file.

#### Model Specific Connections

You can specify a custom connection for a model by setting the `$connection` property:

```php
namespace App\Models;

use Database\BaseModel;

class AuditLog extends BaseModel
{
    protected string $connection = 'audit';
}
```

Queries for this model will now automatically use the `audit` connection.

#### Common Use Cases

- **Data Archiving**: Move large, rarely accessed tables like `AuditLog` or `History` to a separate, cheaper database (e.g., an external SQLite file).
- **Reporting & Analytics**: Direct reporting models to a read-only replica to prevent complex analytics queries from slowing down your primary transactional database.
- **Microservices Hybrid**: If your application shares a database with another service, you can map specific models to that external database while keeping your app's core data local.
- **Legacy Integration**: Map models to a legacy database for gradual migration or integration.

### Accessing Low-Level Objects

```php
$pdo = DB::connection()->getPdo();
$driver = DB::connection()->getDriver(); // 'mysql', 'sqlite', etc.
$config = DB::connection()->getConfig();
```

#### Examples

**User Registration**

```php
use Database\DB;

DB::transaction(function() use ($data) {
    // Create user
    $userId = DB::table('users')->insertGetId([
        'name' => $data['name'],
        'email' => $data['email'],
        'password' => enc()->hashPassword($data['password']),
        'created_at' => datetime()->now(),
    ]);

    // Create profile
    DB::table('profiles')->insert([
        'user_id' => $userId,
        // ...
    ]);
});
```

**Complex Reports**

```php
// Aggregates
$stats = DB::table('posts')
    ->select([
        DB::raw('COUNT(*) as total'),
        DB::raw('AVG(views) as avg_views')
    ])
    ->first();
```

## Best Practices

1. **Prioritize Query Builder**: It is safer and cleaner than raw SQL.
2. **Use Transactions**: Ensure data integrity for multi-step operations.
3. **Use Prepared Statements**: The framework does this automatically for bindings.
4. **Monitor Performance**: Use `whenQueryingForLongerThan` in production.
5. **Index Columns**: Add database indexes for widely used headers.

## Available Methods Summary

## DB Facade Reference

#### table

```php
table(string $table): Builder
```

Returns a Query Builder instance for the specified table.

- **Use Case**: The entry point for all fluent database queries.
- **Example**: `DB::table('users')->get();`.

#### select

```php
select(string $table, array $columns = ['*']): array
```

A shortcut method to perform a simple SELECT query.

- **Example**: `DB::select('users', ['id', 'email']);`.

#### insert

```php
insert(string $table, array $values): bool
```

Inserts a new record (or multiple records) into the table.

- **Use Case**: Basic row creation.
- **Example**: `DB::insert('logs', ['action' => 'login', 'user_id' => 1]);`.

#### update

```php
update(string $table, array $values): int
```

Updates records in the table. Returns the number of affected rows.

- **Note**: Usually preceded by a `where()` clause if called via `table()`.

#### delete

```php
delete(string $table): int
```

Deletes records from the table. Returns the number of affected rows.

#### transaction

```php
transaction(callable $callback): mixed
```

Executes a closure within a database transaction. Automatically rolls back if an exception occurs.

- **Use Case**: Critical multi-table operations (e.g., Transferring money between accounts).

#### raw

```php
raw(string $expression, array $bindings = []): RawExpression
```

Creates a raw SQL expression that won't be escaped by the query builder.

- **Caution**: Use with care to prevent SQL injection. Always use the `$bindings` parameter for user input.
