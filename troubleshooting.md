# Troubleshooting Guide

Quick solutions to common issues in the Anchor Framework.

## Installation Issues

### Composer Dependency Conflicts

**Error:** `Your requirements could not be resolved to an installable set of packages`

**Solution:**

```bash
# Clear composer cache
composer clear-cache

# Update dependencies
composer update

# If still failing, remove vendor and reinstall
rm -rf vendor
composer install
```

### PHP Version Requirements

**Error:** `requires php ^8.1 but your php version is 7.4`

**Solution:**

- Upgrade PHP to 8.1 or higher
- Update `composer.json` if you need to support older PHP versions
- Check PHP version: `php -v`

### Missing PHP Extensions

**Error:** `ext-pdo is missing from your system`

**Solution:**

```bash
# Ubuntu/Debian
sudo apt-get install php8.1-pdo php8.1-mysql php8.1-sqlite3

# macOS (Homebrew)
brew install php@8.1

# Windows
# Enable extensions in php.ini:
extension=pdo_mysql
extension=pdo_sqlite
```

## Database Issues

### Connection Failed

**Error:** `SQLSTATE[HY000] [2002] Connection refused`

**Causes:**

- Database server not running
- Wrong credentials in `.env`
- Incorrect host/port

**Solution:**

```bash
# 1. Check database server is running
# MySQL
sudo service mysql status

# 2. Verify .env credentials
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=your_database
DB_USERNAME=your_username
DB_PASSWORD=your_password

# 3. Test connection
php dock database:tables
```

### Migration Errors

**Error:** `Table 'migrations' doesn't exist`

**Solution:**

```bash
# Create migrations table
php dock migration:run

# If migrations table is corrupted
php dock database:create migrations
```

**Error:** `SQLSTATE[42S01]: Base table or view already exists`

**Solution:**

```bash
# Reset migrations
php dock migration:reset

# Or rollback and re-run
php dock migration:rollback
php dock migration:run
```

### Query Builder Issues

**Error:** `Call to undefined method where()`

**Solution:**

```php
// Wrong - calling static method on model class
User::where('email', $email);

// Correct - use query builder via DB facade
DB::table('user')->where('email', $email)->first();

// Or use model query() method
User::query()->where('email', $email)->first();

// Or use model helper methods
User::find($id);
User::findByEmail($email);
```

## Queue Issues

### Worker Not Processing Jobs

**Problem:** Jobs stay in `pending` status

**Solution:**

```bash
# 1. Check worker status
php worker status

# 2. Start worker if not running
php worker start

# 3. Check queue status
php dock queue:check

# 4. Verify job class exists
# Make sure the job class is autoloaded
composer dump-autoload
```

### Jobs Failing Silently

**Problem:** Jobs marked as `failed` with no error

**Solution:**

```bash
# 1. Check error logs (in root directory)
tail -f error.log

# 2. Run job manually to see error
# (No direct command to run a single job yet, rely on logs)

# 3. Add logging to job
public function handle() {
    logger('queue.log')->info('Job started');
    // ... job logic
    logger('queue.log')->info('Job completed');
}
```

### Memory Exhaustion

**Error:** `Allowed memory size exhausted`

**Solution:**

```bash
# 1. Increase PHP memory limit
php -d memory_limit=512M worker start

# 2. Or update php.ini
memory_limit = 512M

# 3. Process jobs in smaller batches
# Avoid loading too much data at once
```

### Queue Pause Not Working

**Problem:** `php dock queue:pause` doesn't stop processing

**Solution:**

```bash
# 1. Check worker is running
php dock queue:check

# 2. Force stop worker
php worker stop

# 3. Restart worker
php worker start

# 4. Verify pause status
php dock queue:check
```

## Performance Issues

### Slow Queries

**Problem:** Database queries taking too long

**Solution:**

```php
use Database\Connection;

// 1. Query logging is automatic - just run your code
$users = DB::table('user')->get();

// 2. Check logged queries
$queries = Connection::getQueryLog();
foreach ($queries as $query) {
    echo $query['sql'] . ' - ' . $query['time_ms'] . 'ms' . PHP_EOL;
}

// 3. Add indexes to improve performance
Schema::table('user', function ($table) {
    $table->index('email');
    $table->index(['status', 'created_at']);
});
```

### N+1 Query Problem

**Problem:** Too many database queries

**Solution:**

```php
// N+1 Problem
$users = User::query()->get();
foreach ($users as $user) {
    echo $user->profile->bio; // Separate query for each user
}

// Solution: Eager loading
$users = User::query()->with('profile')->get();
foreach ($users as $user) {
    echo $user->profile->bio; // No additional queries
}
```

### Cache Not Working

**Problem:** Cache always returns null

**Solution:**

```bash
# 1. Check cache directory permissions
chmod -R 775 App/storage/cache

# 2. Clear cache
php dock cache:flush

# 3. Verify cache is enabled
# Check App/Config/cache.php
'enabled' => true,
```

## Testing Issues

### Tests Failing on SQLite

**Error:** `SQLite does not support this operation`

**Solution:**

```php
// Use skipOnSqlite() helper
test('foreign key constraints work', function () {
    skipOnSqlite('Foreign keys handled differently');

    // Test code...
});
```

### Database State Not Resetting

**Problem:** Tests fail due to data from previous tests

**Solution:**

```php
// Use beforeEach to reset state
beforeEach(function () {
    Schema::dropIfExists('test_table');
    Schema::create('test_table', function ($table) {
        $table->id();
        // ...
    });
});

afterEach(function () {
    Schema::dropIfExists('test_table');
});
```

### Test Isolation Issues

**Problem:** Tests pass individually but fail when run together

**Solution:**

```php
// 1. Use separate database connections
beforeEach(function () {
    $this->connection = Connection::configure('sqlite::memory:')
        ->name('test_' . uniqid())
        ->connect();
});

// 2. Clean up after each test
afterEach(function () {
    $this->connection = null;
});
```

## Deployment Issues

### Environment Configuration

**Problem:** App works locally but fails in production

**Solution:**

```bash
# 1. Copy .env.example to .env
cp .env.example .env

# 2. Set production values
APP_ENV=production
APP_DEBUG=false
APP_URL=yourdomain.com

# 3. Generate app key
php dock key:generate

# 4. Clear cache
php dock cache:flush
```

### File Permissions

**Error:** `Permission denied` when writing files

**Solution:**

```bash
# Set correct permissions
chmod -R 775 App/storage
chmod -R 775 public/uploads

# Set correct ownership
chown -R www-data:www-data App/storage
chown -R www-data:www-data public/uploads
```

### .htaccess Not Working

**Problem:** 404 errors on all routes

**Solution:**

```bash
# 1. Enable mod_rewrite
sudo a2enmod rewrite

# 2. Update Apache config
<Directory /var/www/html>
    AllowOverride All
</Directory>

# 3. Restart Apache
sudo service apache2 restart

# 4. Verify .htaccess exists in project root
```

### Production Errors

**Error:** White screen / 500 error

**Solution:**

```bash
# 1. Check error logs (in root directory)
tail -f error.log

# 2. Enable error display temporarily
# In .env
APP_DEBUG=true

# 3. Check PHP error log
tail -f /var/log/php/error.log

# 4. Verify all dependencies installed
composer install --no-dev --optimize-autoloader
```

## CLI Command Issues

### Command Not Found

**Error:** `Command "test" is not defined`

**Solution:**

```bash
# 1. Clear cache
php dock cache:flush

# 2. Verify command is registered in App/Config/command.php

# 3. Ensure command class exists:
# - System commands: System/Cli/Commands/
# - User commands: App/Commands/

# 4. Regenerate autoload
composer dump-autoload
```

### Sail Command Fails

**Error:** `Binary not found: pint`

**Solution:**

```bash
# Install missing dependencies
composer require laravel/pint --dev
composer require phpstan/phpstan --dev
composer require pestphp/pest --dev

# Verify installation
vendor/bin/pint --version
vendor/bin/phpstan --version
vendor/bin/pest --version
```

## Getting Help

If you're still stuck after trying these solutions:

- **Check Documentation**: [README](README.md)
- **Search Issues**: Look for similar problems in GitHub Issues
- **Enable Debug Mode**: Set `APP_DEBUG=true` in `.env`
- **Check Logs**: Review `error.log` in the root directory
- **Ask for Help**: Open a GitHub Issue with:
  - Error message
  - Steps to reproduce
  - PHP version (`php -v`)
  - Framework version
  - Relevant code snippets

## Quick Reference

### Common Commands

```bash
# Database
php dock database:create <name>
php dock migration:run
php dock migration:rollback
php dock seeder:run

# Queue
php dock queue:check
php dock queue:pause
php dock queue:resume
php worker start
php worker status
php worker stop

# Testing
php dock test
php dock test --filter=UserTest
php dock test --coverage

# Code Quality
php dock inspect
php dock format
php dock sail

# Cache
php dock cache:flush
```

### Useful Debugging Functions

```php
// Dump and die
dd($variable);

// Dump without dying
dump($variable);

// Log to file (requires filename parameter)
logger('debug.log')->info('Debug message', ['data' => $data]);

// Query logging (automatic)
use Database\Connection;

// ... run queries
dd(Connection::getQueryLog());
```
