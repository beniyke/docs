# Query Builder

Anchor's **Query Builder** provides a fluent, expressive API for constructing SQL queries across MySQL, PostgreSQL, SQLite, and MariaDB. It lives in `Database\Query\Builder` and is accessed via the `DB` facade or directly through a `Connection` instance.

## Getting Started

```php
use Database\DB;

// Using the DB facade (recommended)
$users = DB::table('users')
    ->where('active', '=', 1)
    ->get();

// Or via a Connection instance
$conn = DB::connection();
$users = $conn->table('users')->where('active', '=', 1)->get();
```

- `DB::table(string $table)` returns a fresh `Builder` bound to the default connection.
- You can also call `DB::connection()->table('users')` for explicit connection handling.

## Basic SELECT Queries

```php
// Select all columns (default)
$all = DB::table('posts')->get();

// Select specific columns
$titles = DB::table('posts')->select(['id', 'title'])->get();

// Retrieve the first record
$first = DB::table('posts')->where('id', '=', 1)->first();

// Retrieve a record by its primary key
$post = DB::table('posts')->find(1);

// Get a single column value
$count = DB::table('posts')->count();

// Get a specific value from the first row
$email = DB::table('users')->where('id', 1)->value('email');

// Get a list of column values
$titles = DB::table('posts')->pluck('title');
```

### Method Reference

| Method                                          | Description                              |
| :---------------------------------------------- | :--------------------------------------- |
| `select(array $columns = ['*'])`                | Set the column list.                     |
| `selectRaw(string $expr, array $bindings = [])` | Add a raw select expression.             |
| `distinct()`                                    | Add `DISTINCT` to the query.             |
| `from(string\|Builder $table)`                  | Set the table to query from.             |
| `first(array $columns = ['*'])`                 | Fetch the first row.                     |
| `find(mixed $id, array $columns = ['*'])`       | Fetch a record by primary key.           |
| `get(array $columns = ['*'])`                   | Execute and return a collection.         |
| `count()`                                       | Return the row count.                    |
| `exists()`                                      | Boolean existence check.                 |
| `doesntExist()`                                 | Negated existence check.                 |
| `value(string $column)`                         | Fetch a single value from the first row. |
| `rawValue(string $column)`                      | Fetch a single value as a raw string.    |
| `pluck(string $column)`                         | Fetch an array of values for a column.   |

## WHERE Clauses

### Basic Conditions

```php
$users = DB::table('users')
    ->where('age', '>', 18)
    ->where('status', '=', 'active')
    ->get();
```

| Method                                     | Description                             |
| :----------------------------------------- | :-------------------------------------- |
| `where($column, $operator = '=', $value)`  | Basic comparison.                       |
| `orWhere(...)`                             | Same as `where` but combined with `OR`. |
| `whereNotEqual($column, $value)`           | Condition `!=`.                         |
| `whereLike($column, $value)`               | Condition `LIKE`.                       |
| `whereNotLike($column, $value)`            | Condition `NOT LIKE`.                   |
| `orWhereLike(...)`                         | OR Condition `LIKE`.                    |
| `orWhereNotLike(...)`                      | OR Condition `NOT LIKE`.                |
| `whereGreaterThan($column, $value)`        | Condition `>`.                          |
| `whereLessThan($column, $value)`           | Condition `<`.                          |
| `whereGreaterThanOrEqual($column, $value)` | Condition `>=`.                         |
| `whereLessThanOrEqual($column, $value)`    | Condition `<=`.                         |
| `whereAnyLike(array $cols, $pattern)`      | Multi-column `LIKE` search.             |
| `orWhereAnyLike(...)`                      | OR Multi-column `LIKE` search.          |
| `whereRegexp($column, $pattern)`           | Matches a regular expression.           |
| `whereMatch($cols, $val)`                  | Full-Text search (MySQL/PostgreSQL).    |

### Nested Conditions

```php
$users = DB::table('users')
    ->where(function($q) {
        $q->where('role', '=', 'admin')
          ->orWhere('role', '=', 'editor');
    })
    ->where('active', '=', 1)
    ->get();
```

| Method                         | Description                      |
| :----------------------------- | :------------------------------- |
| `where(callable $callback)`    | Creates a nested `AND` group.    |
| `orWhere(callable $callback)`  | Creates a nested `OR` group.     |
| `whereNot(callable $callback)` | Wraps the nested group in `NOT`. |

### JSON Queries

JSON Queries allow you to inspect and filter data stored in JSON columns. This is useful for working with semi-structured data like user preferences or configuration settings without needing separate tables.

```php
$users = DB::table('users')
    ->whereJsonContains('metadata', '$.preferences.newsletter', true)
    ->get();
```

| Method                                                              | Description                        |
| :------------------------------------------------------------------ | :--------------------------------- |
| `whereJsonContains($col, $path, $val, $bool = 'AND', $not = false)` | Check if JSON path contains value. |
| `whereJsonDoesntContain(...)`                                       | Negated version.                   |
| `whereJsonLength($col, $path, $op, $val)`                           | Check JSON array length.           |

### Date & Time Queries

```php
$orders = DB::table('orders')
    ->whereDate('created_at', '>=', '2024-01-01')
    ->whereYear('created_at', 2024)
    ->whereMonthBetween('created_at', 1, 3) // Q1
    ->get();
```

| Method                                                                | Description                                                |
| :-------------------------------------------------------------------- | :--------------------------------------------------------- |
| `whereDate(string $column, string $operator, string $value)`          | DATE comparison.                                           |
| `whereMonth(string $column, string $operator, int $value)`            | MONTH comparison.                                          |
| `whereYear(string $column, int $value)`                               | YEAR comparison.                                           |
| `whereDay(string $column, string $operator, int $value)`              | DAY comparison.                                            |
| `whereMonthBetween(string $column, int $start, int $end)`             | Month range check.                                         |
| `whereDayBetween(string $column, string $type, int $start, int $end)` | Day-of-week/month/year ranges.                             |
| `whereOnOrAfter(string $column, mixed $date)`                         | Date >= value.                                             |
| `whereOnOrBefore(string $column, mixed $date)`                        | Date <= value.                                             |
| `whereAfter(string $column, mixed $date)`                             | Date > value.                                              |
| `laterThan(string $column, mixed $date)`                              | Alias for whereAfter.                                      |
| `furtherThan(string $column, mixed $date)`                            | Alias for whereAfter.                                      |
| `newerThan(string $column, mixed $date)`                              | Alias for whereAfter.                                      |
| `whereBefore(string $column, mixed $date)`                            | Date < value.                                              |
| `olderThan(string $column, mixed $date)`                              | Alias for whereBefore.                                     |
| `earlierThan(string $column, mixed $date)`                            | Alias for whereBefore.                                     |
| `upTo(string $column, mixed $date)`                                   | Alias for whereOnOrBefore.                                 |
| `since(string $column, mixed $date)`                                  | Alias for whereOnOrAfter.                                  |
| `whereLast(string $unit, int $value, string $column = 'created_at')`  | Query for last X days/months/etc.                          |
| `whereBetween(string $column, array [$min, $max])`                    | Generic range with AND.                                    |
| `orWhereBetween(string $column, array [$min, $max])`                  | Generic range with OR.                                     |
| `whereNotBetween(string $column, array [$min, $max])`                 | Negated range.                                             |
| `whereDateBetween(string $column, array [$start, $end])`              | Date range.                                                |
| `whereIn(string $column, array $values)`                              | Value in list. Empty array results in a `FALSE` condition. |
| `whereIntegerInRaw(string $col, array $vals)`                         | Optimized integer list (no bindings).                      |
| `whereNotIn(string $column, array $values)`                           | Value not in list. Empty array adds no condition.          |
| `whereNull(string $column)`                                           | Checks if column is NULL.                                  |
| `whereNotNull(string $column)`                                        | Checks if column is NOT NULL.                              |
| `whereColumn(string $first, string $operator, string $second)`        | Compare values of two columns.                             |
| `whereBelongsTo($modelOrId, $key)`                                    | Filter by related model.                                   |
| `whereRelation($table, $key, $col, $val)`                             | Filter by relationship condition.                          |

**Example with `orWhereBetween`:**

```php
// Find records where start_date OR end_date falls within a range
$schedules = DB::table('schedules')
    ->where(function($q) use ($start, $end) {
        $q->whereBetween('starts_at', [$start, $end])
          ->orWhereBetween('ends_at', [$start, $end]);
    })
    ->get();
```

**Raw Expressions**

```php
$users = DB::table('users')
    ->whereRaw('DATEDIFF(NOW(), last_login) > ?', [30])
    ->get();
```

| Method                                                                  | Description                    |
| :---------------------------------------------------------------------- | :----------------------------- |
| `whereRaw(string $sql, array $bindings = [], string $boolean = 'AND')`  | Add a raw `WHERE` clause.      |
| `selectRaw(string $expression, array $bindings = [])`                   | Add a raw `SELECT` expression. |
| `havingRaw(string $sql, array $bindings = [], string $boolean = 'AND')` | Add a raw `HAVING` clause.     |

> **Note:** For OR behavior, pass `'OR'` as the `$boolean` argument to `whereRaw` or `havingRaw`.

## JOINs

JOINs allow you to query data from multiple tables in a single request. Use them when you need to fetch related data (e.g., a post and its author) efficiently without executing N+1 queries.

```php
$posts = DB::table('posts')
    ->join('users', 'posts.user_id', '=', 'users.id')
    ->leftJoin('categories', function($join) {
        $join->on('posts.category_id', '=', 'categories.id')
             ->where('categories.active', '=', 1);
    })
    ->select(['posts.*', 'users.name as author'])
    ->get();
```

| Method                                                             | Description                           |
| :----------------------------------------------------------------- | :------------------------------------ |
| `join($table, $first, $op = '=', $second = null, $type = 'inner')` | SQL JOIN. `$first` can be a callable. |
| `leftJoin(...)`                                                    | Shortcut for LEFT JOIN.               |
| `rightJoin(...)`                                                   | Shortcut for RIGHT JOIN.              |

## Aggregations & Grouping

Aggregations allow you to compute values (like counts, sums, or averages) directly in the database. Grouping helps you categorize these calculated values by a specific column (e.g., sales per month). Use these features for reporting and statistical summaries.

**Key Difference:** Use `WHERE` to filter rows _before_ grouping, and `HAVING` to filter groups _after_ aggregation. For example, `WHERE` finds "orders from 2024", while `HAVING` finds "months with more than 100 orders".

```php
$stats = DB::table('orders')
    ->select([
        DB::raw('COUNT(*) as total'),
        DB::raw('SUM(amount) as revenue'),
        DB::raw('AVG(amount) as avg_order')
    ])
    ->groupBy('status')
    ->having('total', '>', 100)
    ->get();
```

| Method                                     | Description                          |
| :----------------------------------------- | :----------------------------------- |
| `groupBy($columns)`                        | GROUP BY clause.                     |
| `having($column, $op, $val)`               | HAVING clause.                       |
| `orHaving(...)`                            | OR HAVING clause.                    |
| `havingRaw($sql, $bindings = [])`          | Raw HAVING clause.                   |
| `withCount($relations, $alias = null)`     | Eager-load count of a relationship.  |
| `withSum`, `withAvg`, `withMin`, `withMax` | Aggregate helpers for relationships. |
| `min($column)`                             | Get minimum value.                   |
| `max($column)`                             | Get maximum value.                   |
| `sum($column)`                             | Get sum of values.                   |
| `avg($column)`                             | Get average value.                   |

## Ordering, Limits & Offsets

Ordering controls the sequence of your results, while limits and offsets allow you to fetch a subset of data. These are essential for building features like pagination, "Top 10" lists, or simply ensuring data is presented in a logical order.

```php
$users = DB::table('users')
    ->orderBy('created_at', 'desc')
    ->orderBy('name')
    ->limit(20)
    ->offset(40) // page 3 (20 per page)
    ->get();
```

| Method                           | Description             |
| :------------------------------- | :---------------------- |
| `orderBy($column, $dir = 'asc')` | Order by column.        |
| `latest($column = 'created_at')` | Order by descending.    |
| `oldest($column = 'created_at')` | Order by ascending.     |
| `inRandomOrder()`                | Order results randomly. |
| `limit(int $limit)`              | Limit results.          |
| `take(int $limit)`               | Alias for limit.        |
| `offset(int $offset)`            | Offset results.         |
| `skip(int $offset)`              | Alias for offset.       |

## Unions & CTEs

**Unions** allow you to combine the result sets of two or more queries into a single result set. This is useful when you need to aggregate data from different tables or conditions that have the same structure.

**Common Table Expressions (CTEs)** provide a way to name a temporary result set that exists within the scope of a single statement. They are powerful for breaking down complex queries, improving multiple references to the same data, or handling hierarchical data structures (like category trees) using recursive logic.

```php
$union = DB::table('users')
    ->where('role', '=', 'admin')
    ->union(
        DB::table('users')->where('role', '=', 'editor')
    )
    ->get();

// Common Table Expression (CTE)
// Recursive CTE
$cte = DB::table('orders')
    ->withRecursive('order_tree', function($query) {
        $query->select(['id', 'parent_id'])
              ->from('orders')
              ->whereNull('parent_id');
    })
    ->select('*')
    ->from('order_tree')
    ->get();

// Non-recursive CTE
$result = DB::table('users')
    ->withoutRecursive('active_users', function($query) {
        $query->select(['id', 'name'])
              ->from('users')
              ->where('active', '=', 1);
    })
    ->select('*')
    ->from('active_users')
    ->get();
```

| Method                            | Description                      |
| :-------------------------------- | :------------------------------- |
| `union(Builder $query)`           | UNION (distinct rows).           |
| `unionAll(Builder $query)`        | UNION ALL (includes duplicates). |
| `withRecursive($name, $query)`    | Adds a recursive CTE.            |
| `withoutRecursive($name, $query)` | Adds a non-recursive CTE.        |

> **Note:** `with(string|array $relations)` is for eager loading relationships, not CTEs.

## Pagination

Pagination splits large datasets into smaller, manageable chunks (pages) for display in a UI. Anchor supports both offset-based pagination (standard page numbers) and cursor-based pagination (infinite scrolling or high-performance APIs).

### Offset-Based Pagination

```php
$page = 2;
$perPage = 25;

$paginator = DB::table('posts')
    ->orderBy('created_at', 'desc')
    ->paginate($perPage, $page);

// $paginator is an instance of `Database\Pagination\Paginator`
```

| Method                                              | Description              |
| :-------------------------------------------------- | :----------------------- |
| `paginate($perPage = 15, $page = 1)`                | Offset-based pagination. |
| `forPage($page, $perPage = 15)`                     | Manual paging limits.    |
| `isPageValid($page, $perPage)`                      | Validate page number.    |
| `cursor($after = null, $perPage = 15, $col = 'id')` | Cursor-based pagination. |

### Cursor-Based Pagination

```php
// First page
$paginator = DB::table('posts')
    ->setModelClass(Post::class)
    ->cursor(null, 25, 'id');

// Next page (using the last item's ID as cursor)
$lastId = $paginator->getNextCursor();
$nextPage = DB::table('posts')
    ->setModelClass(Post::class)
    ->cursor($lastId, 25, 'id');
```

## Caching & Query Logging

Query caching improves performance by storing the result of a query for a specified duration, preventing repeated database hits. Query logging is useful for debugging to see exactly what SQL is being executed and how long it takes.

### Prepared‑Statement Caching

The `Builder` automatically uses the underlying `Connection`'s prepared‑statement cache. You can control it via the connection:

```php
$conn = DB::connection();
$conn->setMaxCacheSize(200); // increase cache size
$conn->clearStatementCache(); // clear manually
```

### Query Cache

Query cache stores the result of a query for a specified duration, preventing repeated database hits. Result caching is useful for queries that are executed frequently and return the same result set.

```php
$users = DB::table('users')
    ->where('active', '=', 1)
    ->cache(60) // cache result for 60 seconds
    ->get();
```

| Method                                 | Description                        |
| :------------------------------------- | :--------------------------------- |
| `cache(int $seconds = 3600)`           | Cache the result set.              |
| `cacheTags(array $tags)`               | Assign cache tags.                 |
| `cacheWithStale(int $seconds = 3600)`  | Cache with stale-while-revalidate. |
| `clearQueryLog()`                      | Reset the log.                     |
| `toSql()`                              | Get SQL string.                    |
| `getBindings()`                        | Get query bindings.                |
| `flushQueryCache()`                    | Manually flush query cache.        |
| `whenQueryingForLongerThan($sec, $cb)` | Slow-query detection.              |

### Query Log

```php
$log = DB::connection()->getQueryLog();
foreach ($log as $entry) {
    echo "SQL: {$entry['sql']} | Time: {$entry['time_ms']}ms\n";
}
```

## Soft Deletes & Global Scopes

### Soft Deletes

Models using the `SoftDeletes` trait automatically add a `whereNull('deleted_at')` clause. You can disable it per‑query:

```php
$users = DB::table('users')
    ->withoutSoftDeletes()
    ->get();

// Restore soft-deleted records
DB::table('users')->where('id', 1)->restore();
```

### Global Scopes

Register a global scope once per model:

```php
DB::table('orders')
    ->addGlobalScope('active', function($query) {
        $query->where('status', '=', 'active');
    })
    ->get();
```

| Method                                         | Description                    |
| :--------------------------------------------- | :----------------------------- |
| `withoutGlobalScope(string $identifier)`       | Remove a specific scope.       |
| `withoutGlobalScopes(array $identifiers = [])` | Remove all or selected scopes. |

## Model Integration & Scopes

When a model class is attached via `setModelClass('App\Models\User')`, the builder forwards unknown method calls to the model’s **local scopes**:

```php
class User extends BaseModel {
    public function scopeRecent(Builder $query, int $days = 30) {
        return $query->where('created_at', '>=', datetime()->subDays($days));
    }
}

$users = DB::table('users')
    ->setModelClass(User::class)
    ->recent(15) // calls User::scopeRecent
    ->get();
```

- Scopes must be prefixed with `scope` in the model.
- The builder automatically injects itself as the first argument.

### Eager Loading

```php
$users = DB::table('users')
    ->setModelClass(User::class)
    ->with(['posts', 'profile'])
    ->get();
```

- `with(string|array $relations)` - eager load relationships (requires model class).

## Transactions

Transactions allow you to execute a group of database operations as a single unit. If any operation fails, the entire group matches rolled back, ensuring your database remains in a consistent state. Use transactions for critical operations like money transfers or multi-step account creation.

```php
DB::transaction(function() {
    DB::table('accounts')->where('id', '=', 1)->decrement('balance', 100);
    DB::table('accounts')->where('id', '=', 2)->increment('balance', 100);
});
```

| Method                          | Description                  |
| :------------------------------ | :--------------------------- |
| `DB::transaction(callable $cb)` | Run callback in transaction. |
| `DB::beginTransaction()`        | Manually start transaction.  |
| `DB::commit()`                  | Manually commit.             |
| `DB::rollBack()`                | Manually rollback.           |
| `DB::afterCommit($cb)`          | Register commit callback.    |
| `DB::afterRollback($cb)`        | Register rollback callback.  |

## Conditional Clauses

Sometimes you may want to apply a clause only if a condition is true:

```php
$role = request('role');

$users = DB::table('users')
    ->when($role, function ($query, $role) {
        return $query->where('role_id', $role);
    })
    ->get();
```

| Method                                                              | Description                 |
| :------------------------------------------------------------------ | :-------------------------- |
| `when(mixed $value, callable $callback, ?callable $default = null)` | Apply clause conditionally. |

## Locking

```php
// Shared lock (SELECT ... LOCK IN SHARE MODE)
DB::table('users')->where('votes', '>', 100)->lockForSharedReading()->get();

// Update lock (SELECT ... FOR UPDATE)
DB::table('users')->where('votes', '>', 100)->lockForUpdate()->get();
```

| Method                   | Description                                  |
| :----------------------- | :------------------------------------------- |
| `lockForSharedReading()` | Shared lock (SELECT ... LOCK IN SHARE MODE). |
| `lockForUpdate()`        | Update lock (SELECT ... FOR UPDATE).         |

## Advanced Helpers

### Raw Expressions

```php
$users = DB::table('users')
    ->select([DB::raw('COUNT(*) as total'), DB::raw('MAX(score) as max_score')])
    ->get();
```

- `DB::raw(string $expression)` – creates a raw SQL fragment.
- Use in `select`, `whereRaw`, etc.

### Sub‑queries

```php
$users = DB::table('users')
    ->whereIn('id', function($q) {
        $q->select('user_id')
          ->from('orders')
          ->where('status', '=', 'completed');
    })
    ->get();
```

| Method | Description |
| :- | : | |
| `whereIn(string $column, Builder | array $values)` | Supports sub‑queries. If an empty array is passed, it results in a `1 = 0` (always false) condition. |
| `whereNotIn(string $column, Builder | array $values)` | If an empty array is passed, it adds no condition to the query. |
| `whereExists`, `whereNotExists` | Existence checks with sub‑queries. |

## Inserts

```php
// Single insert
DB::table('users')->insert([
    'email' => 'john@example.com',
    'votes' => 0
]);

// Multiple insert
DB::table('users')->insert([
    ['email' => 'taylor@example.com', 'votes' => 0],
    ['email' => 'dayle@example.com', 'votes' => 0]
]);

// Insert and get ID
$id = DB::table('users')->insertGetId(
    ['email' => 'john@example.com', 'votes' => 0]
);

// Insert or ignore (database dependent)
DB::table('users')->insertOrIgnore([
    'id' => 1,
    'email' => 'john@example.com'
]);
```

## Updates & Deletes

### Updates

```php
// Basic update
DB::table('users')
    ->where('id', 1)
    ->update(['votes' => 1]);

// Increment & Decrement
DB::table('users')->increment('votes');     // +1
DB::table('users')->increment('votes', 5);  // +5
DB::table('users')->decrement('votes');     // -1

// Increment/Decrement Multiple Columns
DB::table('users')->incrementEach([
    'votes' => 1,
    'balance' => 100
]);

DB::table('users')->decrementEach([
    'votes' => 1,
    'balance' => 50
]);
```

### Deletes

```php
// Delete records
DB::table('users')->where('votes', '<', 100)->delete();

// Truncate table
DB::table('users')->truncate();
```

## Chunking Results

For processing large datasets, use `chunk` to conserve memory:

```php
DB::table('users')->orderBy('id')->chunk(100, function ($users) {
    foreach ($users as $user) {
        // Process user...
    }
});

// Stop processing by returning false
DB::table('users')->orderBy('id')->chunk(100, function ($users) {
    // ...
    return false;
});
```

## Error Handling

All builder methods throw `RuntimeException` for misuse (e.g., missing table) and `InvalidArgumentException` for invalid arguments.

```php
try {
    DB::table('nonexistent')->get();
} catch (RuntimeException $e) {
    logger('query.log')->error('Query error: ' . $e->getMessage());
}
```

**Examples**

**Paginated Blog**

```php
$page = request()->get('page', 1);
$perPage = 10;

$posts = DB::table('posts')
    ->where('published', '=', 1)
    ->orderBy('published_at', 'desc')
    ->paginate($perPage, $page);

return view('blog.index', ['posts' => $posts]);
```

**Complex Reporting Query**

```php
$report = DB::table('orders')
    ->select([
        DB::raw('DATE(created_at) as day'),
        DB::raw('COUNT(*) as orders'),
        DB::raw('SUM(total) as revenue')
    ])
    ->whereBetween('created_at', ['2024-01-01', '2024-01-31'])
    ->groupBy('day')
    ->orderBy('day')
    ->cacheFor(300) // cache for 5 minutes
    ->get();
```

**Eager‑Loading Relationship Counts**

```php
$users = DB::table('users')
    ->withCount('posts')
    ->withSum('orders', 'amount')
    ->get();
```

## Summary

The Anchor Query Builder is a **feature‑rich**, **fluent** API that abstracts away SQL quirks while still giving you full control when needed. It supports:

- All basic CRUD operations.
- Rich `WHERE` capabilities (nested, JSON, date, raw).
- Joins, unions, CTEs, and sub‑queries.
- Aggregations, grouping, having, and eager‑load aggregates.
- Pagination (offset/limit and cursor based).
- Prepared‑statement caching and result caching.
- Soft deletes, global scopes, and model scopes.
- Transaction management and callbacks.
- Error handling with clear exceptions.
