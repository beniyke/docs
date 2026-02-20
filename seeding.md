# Database Seeding

Database seeding allows you to populate your database with test or initial data using seed classes.

## Creating Seeders

Seeders extend `Database\Migration\BaseSeeder` and are stored in your configured seeds directory (default: `App/storage/database/seeds/`):

```php
<?php
use Database\Migration\BaseSeeder;

class UserSeeder extends BaseSeeder
{
    public function run(): void
    {
        // Create test users
        $this->connection->table('user')->insert([
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => enc()->hashPassword('password123'),
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
        ]);

        $this->connection->table('user')->insert([
            'name' => 'Jane Smith',
            'email' => 'jane@example.com',
            'password' => enc()->hashPassword('password123'),
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Generating Seeders

Use the CLI to create a new seeder:

```bash
php dock seeder:create UserSeeder
```

This creates a file in your configured seeds directory with a template ready to use.

## Running Seeders

### Run Specific Seeder

```bash
php dock seeder:run --class=UserSeeder
```

### Run Default Seeder

```bash
php dock seeder:run
```

This runs the `DatabaseSeeder` class by default.

## Calling Other Seeders

You can call other seeders from within a seeder using the `call()` method:

```php
class DatabaseSeeder extends BaseSeeder
{
    public function run(): void
    {
        // Call multiple seeders
        $this->call([
            'UserSeeder',
            'PostSeeder',
            'CommentSeeder',
        ]);
    }
}
```

Or call a single seeder:

```php
$this->call('UserSeeder');
```

## Using the Connection

Access the database connection in your seeder:

```php
class UserSeeder extends BaseSeeder
{
    public function run(): void
    {
        $connection = $this->getConnection();

        // Run raw SQL
        $connection->statement("INSERT INTO user (name, email) VALUES (?, ?)", [
            'John Doe',
            'john@example.com'
        ]);

        // Or use the table method
        $connection->table('user')->insert([
            'name' => 'Jane Smith',
            'email' => 'jane@example.com',
        ]);
    }
}
```

## Examples

### User Seeder with Roles

```php
class UserSeeder extends BaseSeeder
{
    public function run(): void
    {
        // Create admin user
        $this->connection->table('user')->insert([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
            'password' => enc()->hashPassword('admin123'),
            'status' => 'active',
            'role' => 'admin',
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
        ]);

        // Create regular users
        for ($i = 1; $i <= 10; $i++) {
            $this->connection->table('user')->insert([
                'name' => "User {$i}",
                'email' => "user{$i}@example.com",
                'password' => enc()->hashPassword('password123'),
                'status' => 'active',
                'role' => 'user',
                'created_at' => date('Y-m-d H:i:s'),
                'updated_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }
}
```

### Post Seeder with Relationships

```php
class PostSeeder extends BaseSeeder
{
    public function run(): void
    {
        $connection = $this->getConnection();

        // Get all users
        $users = $connection->table('user')->select('id')->get();

        foreach ($users as $user) {
            // Create 5 posts for each user
            for ($i = 1; $i <= 5; $i++) {
                $connection->table('post')->insert([
                    'user_id' => $user->id,
                    'title' => "Post {$i} by User {$user->id}",
                    'content' => 'Lorem ipsum dolor sit amet, consectetur adipiscing elit.',
                    'published' => true,
                    'created_at' => date('Y-m-d H:i:s'),
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
            }
        }
    }
}
```

### Master Seeder

```php
class DatabaseSeeder extends BaseSeeder
{
    public function run(): void
    {
        // Seed in order to respect foreign key dependencies
        $this->call([
            'UserSeeder',
            'CategorySeeder',
            'PostSeeder',
            'CommentSeeder',
            'TagSeeder',
        ]);
    }
}
```

## Seeder Location

Seeders are stored in the directory configured in your database configuration (default: `App/storage/database/seeds/`).

## Best Practices

- **Keep seeders idempotent**: Check if data exists before creating
- **Use transactions**: Wrap in transactions for data integrity
- **Seed in order**: Respect foreign key dependencies
- **Separate test and production**: Use different seeders for different environments
- **Use meaningful data**: Create realistic test data

## Checking Before Seeding

```php
class UserSeeder extends BaseSeeder
{
    public function run(): void
    {
        $connection = $this->getConnection();

        // Only seed if no users exist
        $count = $connection->table('user')->count();

        if ($count === 0) {
            $connection->table('user')->insert([
                'name' => 'Admin',
                'email' => 'admin@example.com',
                'password' => enc()->hashPassword('admin123'),
                'created_at' => date('Y-m-d H:i:s'),
                'updated_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }
}
```

## Using Transactions

The `SeedManager` automatically wraps seeder execution in a transaction, so if any seeder fails, all changes are rolled back:

```php
class UserSeeder extends BaseSeeder
{
    public function run(): void
    {
        // All these inserts happen in a transaction automatically
        $this->connection->table('user')->insert(['name' => 'User 1', 'email' => 'user1@example.com']);
        $this->connection->table('user')->insert(['name' => 'User 2', 'email' => 'user2@example.com']);
        $this->connection->table('user')->insert(['name' => 'User 3', 'email' => 'user3@example.com']);

        // If any insert fails, all are rolled back
    }
}
```

## Error Handling

If a seeder fails, you'll see a detailed error message:

```bash
Seeding FAILED for class UserSeeder. Error: SQLSTATE[23000]: Integrity constraint violation
```

The transaction will be rolled back, and no data will be inserted.
