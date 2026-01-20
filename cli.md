# CLI (Dock)

Anchor includes a powerful command-line interface called `dock`. It provides numerous commands for development, database management, code generation, and more.

## Usage

Run commands from the root of your project:

```bash
php dock [command] [options] [arguments]
```

## Getting Help

```bash
# List all available commands
php dock list

# Get help for a specific command
php dock help [command]

# Example
php dock help migration:run
```

## Global Options

- `-h, --help` - Display help for the command
- `--silent` - Do not output any message
- `-q, --quiet` - Only display errors
- `-V, --version` - Display application version
- `--ansi|--no-ansi` - Force or disable ANSI output
- `-n, --no-interaction` - Do not ask interactive questions
- `-v|vv|vvv, --verbose` - Increase verbosity

## Available Commands

### Development

```bash
# Start PHP server and worker with auto-restart on .env changes
php dock dev

# Interactive playground (REPL) for testing and debugging
php dock playground

# Execute script from _playground directory
php dock playground script_name

# Generate secure APP_KEY
php dock key:generate
```

#### Development Server (`dev`)

The `dev` command is a high-level orchestrator that prepares your local development environment with a single command.

**Lifecycle:**

- **Readiness Check**: Verifies your system meets requirements (PHP version, necessary extensions).
- **Environment Validation**: Ensures a `.env` file exists.
- **Cache Flush**: Automatically flushes the application cache to prevent stale configuration issues.
- **Process Management**: Spawns two persistent background processes:
  - **Built-in Server**: Runs your application at the address defined in `APP_HOST`.
  - **Worker**: Starts the queue worker daemon (`php dock worker:start`).

**Auto-Restart Behavior:**
The command includes a built-in **File Watcher** that monitors your `.env` file for changes. If you update a configuration variable, the `dev` command will:

- Gracefully stop both the Server and Worker processes.
- Flush the application cache again.
- Restart both processes to ensure the new environment variables are active.

> This eliminates the need to manually kill and restart your server every time you tweak a configuration setting.

#### Playground REPL

The playground provides an interactive PHP REPL (Read-Eval-Print Loop) powered by PsySH for testing code, debugging, and exploring your application.

**Features:**

- Full application context (container, database, config, etc.)
- Command history persistence
- Auto-completion
- Helper functions for common tasks
- Production environment protection

**Interactive Mode:**

```bash
php dock playground
```

**Execute Scripts:**

```bash
# Create script in _playground/test.php (without <?php tags)
php dock playground test.php
```

**Available Helpers:**

Framework globals (always available):

- `container()` - Get DI container instance
- `resolve($class)` - Resolve class from container
- `config($key)` - Get configuration value
- `env($key, $default)` - Get environment variable
- `dd(...$args)` - Dump and die
- `dump(...$args)` - Pretty dump

Playground-specific helpers:

- `table($name)` - Quick database table access
- `queries()` - View all executed SQL queries
- `lastQuery()` - View last executed query
- `clearQueries()` - Clear query log
- `clear()` - Clear terminal screen

**Examples:**

```php
// Database queries
table('user')->count()
table('user')->where('active', 1)->get()

// Check container bindings
container()->has('Database\\Connection')

// View configuration
config('app.name')
config('database.connection')

// Test services
$service = resolve(UserService::class)
$service->getAllUsers()

// Debug queries
table('user')->first()
lastQuery()  // See the SQL that was executed
queries()    // See all queries executed in this session
```

#### Advanced Playground Features

**Query Logging**
The playground automatically logs every SQL query executed during your session.

- `lastQuery()`: Displays the most recent query with its bindings.
- `queries()`: Displays an array of all queries executed since the playground started.
- `clearQueries()`: Resets the query log.

**Auto-Imports & Shortcuts**
You can configure the playground to automatically import your most-used classes in `App/Config/playground.php`. This allows you to use `User::find(1)` instead of `\App\Models\User::find(1)`.

**Execution Mode**
You can run the playground in two modes:

1. **Interactive**: (Default) Opens a shell for live interaction.
2. **Scripted**: Pass a filename from the `_playground/` directory (e.g., `php dock playground test.php`) to execute complex testing scripts within the full application context.

**Configuration:**

Edit `App/Config/playground.php` to customize:

- Auto-imports (add frequently-used classes)
- History file location
- Startup message

**Security:**

- Automatically **disabled in production** environments
- No override possible for safety

### Database Management

```bash
# Create a new database
php dock database:create

# Delete a database
php dock database:delete

# List all tables
php dock database:tables

# Truncate tables
php dock database:truncate

# Export database or table
php dock database:export

# Import database or table
php dock database:import
```

### Migrations

```bash
# Create a new migration
php dock migration:create CreateUserTable
# Options: --create=tablename, --table=tablename

# Run pending migrations
php dock migration:run
# Options: --force (for production)

# Rollback last batch
php dock migration:rollback

# Rollback all migrations
php dock migration:reset

# Reset and re-run all migrations
php dock migration:refresh

# List all migrations
php dock migration:list

# Show migration status
php dock migration:status

# Lock migrations (prevent concurrent runs)
php dock migration:lock

# Unlock migrations
php dock migration:unlock
```

### Seeders

```bash
# Create a new seeder
php dock seeder:create UserSeeder

# Run database seeders
php dock seeder:run

# Run specific seeder
php dock seeder:run UserSeeder
```

### Queue Management

```bash
# Check queued jobs by status
php dock queue:check

# Run queued jobs
php dock queue:run

# Flush jobs by status
php dock queue:flush
```

### Worker Management

```bash
# Start queue worker daemon
php dock worker:start

# Stop queue worker daemon
php dock worker:stop

# Restart queue worker daemon
php dock worker:restart

# Check worker status
php dock worker:status
```

### Cache Management

```bash
# Clear all cache or specific namespace
php dock cache:flush [namespace]

# Clear specific cache key
php dock cache:flush [namespace] [key]
```

### Code Generators

Most generators require a **Module Name** as the second argument.

> Many commands automatically append suffixes to the names you provide:
>
> - `controller:create` appends `Controller`
> - `service:create` appends `Service`
> - `action:create` appends `Action`
> - `request:create` appends `Request`
> - `task:create` appends `Task`
> - `*-notification:create` appends `{Type}Notification`
> - `validation:create` appends `{Type}RequestValidation`
>
> For example, `php dock controller:create Tweet Tweets` creates `TweetController`, not `Tweet`.

#### Module

```bash
# Create a new module
php dock module:create Tweet
# Options: --api (skip views)

# Delete a module
php dock module:delete Tweet
```

#### Controller

```bash
# Create a controller (automatically appends 'Controller')
php dock controller:create Tweet Tweet
# Creates: TweetController
# Options: --api (minimal API controller)

# Delete a controller
php dock controller:delete Tweet Tweet
```

#### Model

```bash
# Create a model
php dock model:create Tweet Tweet

# Delete a model
php dock model:delete Tweet Tweet
```

#### Service

```bash
# Create a service (automatically appends 'Service')
php dock service:create Tweet Tweet
# Creates: TweetService

# Delete a service
php dock service:delete Tweet Tweet
```

#### Action

```bash
# Create an action (automatically appends 'Action')
php dock action:create CreateTweet Tweet
# Creates: CreateTweetAction

# Delete an action
php dock action:delete CreateTweet Tweet
```

#### Request

```bash
# Create a request DTO (automatically appends 'Request')
php dock request:create Login Auth
# Creates: LoginRequest

# Delete a request
php dock request:delete Login Auth
```

#### Request Validation

```bash
# Create form validation (--type is required)
php dock validation:create Login Auth --type=form
# Creates: LoginFormRequestValidation in App/src/Auth/Validations/Form/

# Create API validation
php dock validation:create Login Auth --type=api
# Creates: LoginApiApiRequestValidation in App/src/Auth/Validations/Api/

# Delete form validation (--type is required)
php dock validation:delete Login Auth --type=form

# Delete API validation
php dock validation:delete Login Auth --type=api

# Delete multiple validations (comma-separated)
php dock validation:delete Login,Signup Auth --type=form
```

**Note**: The `--type` option is **required** and must be either `form` or `api`. The suffix `{Type}RequestValidation` is automatically appended. You can also use the short form `-t`:

```bash
php dock validation:create Login Auth -t form
```

#### View Components

```bash
# Create view model
php dock view:create-model Tweet Tweet

# Create view template
php dock view:create-template index Tweet

# Create view modal
php dock view:create-modal confirm-delete Tweet

# Delete view model
php dock view:delete-model Tweet Tweet

# Delete view template
php dock view:delete-template index Tweet

# Delete view modal
php dock view:delete-modal confirm-delete Tweet
```

#### Notifications

```bash
# Create email notification (automatically appends 'EmailNotification')
php dock email-notification:create Welcome User
# Creates: WelcomeEmailNotification

# Delete email notification
php dock email-notification:delete Welcome User

# Create in-app notification (automatically appends 'InAppNotification')
php dock inapp-notification:create Login
# Creates: LoginInAppNotification
# Optional: Module name as second argument

# Delete in-app notification
php dock inapp-notification:delete Login
# Optional: Module name as second argument
```

#### Queue Task

```bash
# Create a queue task (automatically appends 'Task')
php dock task:create ProcessOrder Order
# Creates: ProcessOrderTask

# Delete a task
php dock task:delete ProcessOrder Order
```

#### Queue Control

```bash
# Pause queue processing (stops processing jobs, keeps daemon alive)
php dock queue:pause --identifier=default
# If --identifier is omitted, ALL queues are paused

# Resume queue processing
php dock queue:resume --identifier=default
# If --identifier is omitted, ALL paused queues are resumed

# Check queue status
php dock worker:status --queue=default
```

#### Resource

```bash
# Create a resource
php dock resource:create Tweet Tweet

# Delete a resource
php dock resource:delete Tweet Tweet
```

#### Provider

```bash
# Create a service provider
php dock provider:create Tweet Tweet

# Delete a provider
php dock provider:delete Tweet Tweet
```

#### Command

```bash
# Create a CLI command
php dock command:create SendReport

# Delete a command
php dock command:delete SendReport
```

#### Event

```bash
# Create an event class
php dock event:create UserRegistered
# Creates: App/Events/UserRegistered.php

# Create an event in a module
php dock event:create OrderPlaced Shop
# Creates: App/Shop/Events/OrderPlaced.php

# Delete an event
php dock event:delete UserRegistered

# Delete an event from a module
php dock event:delete OrderPlaced Shop
```

#### Listener

```bash
# Create a listener class
php dock listener:create SendWelcomeEmail
# Creates: App/Listeners/SendWelcomeEmail.php

# Create a listener in a module
php dock listener:create UpdateInventory Shop
# Creates: App/Shop/Listeners/UpdateInventory.php

# Delete a listener
php dock listener:delete SendWelcomeEmail

# Delete a listener from a module
php dock listener:delete UpdateInventory Shop
```

### Code Quality & Repair

```bash
# Run code quality checks (Pint + PHPStan)
php dock inspect

# Ultimate production readiness check (CS Fixer + PHPStan + Tests)
php dock sail
```

#### Code Inspection

The `inspect` command runs comprehensive code quality checks:

**What it does:**

1. **Coding Standards (Pint)** - Checks code formatting and style
2. **Static Analysis (PHPStan)** - Analyzes code for type errors and bugs

**Behavior:**

- Runs both checks even if the first one fails
- Reports all issues found
- Returns failure if either check fails

```bash
php dock inspect
```

#### Production Readiness Check (Sail)

The `sail` command is your **ultimate assertion of readiness** before deployment. It runs all quality checks sequentially and fails immediately if any step fails.

**What it does (in order):**

1. **Inspection Check (Zero Errors)**:
   - Coding Standards (Pint)
   - Static Analysis (PHPStan)
2. **Functionality Check (Operational Readiness)** - Full test suite (Pest)

**Behavior:**

- Stops immediately on first failure
- Only succeeds if ALL checks pass
- Validates prerequisites before running

```bash
php dock sail
```

> **Maritime Metaphor**: Just as a ship undergoes final inspections before setting sail, this command ensures your application is ready to "ship" to production. It's your dock's final clearance before the voyage! ⚓🚢

**When to use:**

- Before deploying to production
- Before creating a release
- As part of your CI/CD pipeline
- When you want confidence that everything is production-ready

### Testing

```bash
# Run all tests
php dock test

# Run specific test file
php dock test tests/System/Unit/Helpers/StringTest.php

# Run tests in a directory
php dock test tests/System/Unit/Helpers

# Filter tests by name
php dock test --filter=UserService

# Run tests from a specific group
php dock test --group=database

# Run tests in parallel
php dock test --parallel

# Generate code coverage report
php dock test --coverage

# Create a new test file
php dock test:create UserService

# Create a unit test
php dock test:create StringHelper --unit

# Create a unit test in a specific category
php dock test:create EmailService --unit --category=Mail
```

See the [Testing](testing.md) documentation for comprehensive testing guide.

### System & Environment

```bash
# Downloads and installs/updates the framework core files
php dock anchor:hydrate

# Intelligently updates the framework core
php dock anchor:update

# Encrypt an environment file for version control
php dock env:encrypt

# Decrypt an environment file
php dock env:decrypt

# Generate a new dynamic API key
php dock api:generate-key

# Syncs version from version.txt to System constants
php dock version:sync

# Syncs documentation from docs/ to package READMEs
php dock docs:sync
```

### Maintenance & Auditing

```bash
# Clean up old audit logs
php dock audit:cleanup

# Export audit logs to file
php dock audit:export

# Cleanup old form submissions and logs
php dock stack:cleanup

# Clean up expired OTP codes and records
php dock verify:cleanup

# Display OTP verification statistics
php dock verify:stats
```

### Tenancy

```bash
# Create a new tenant with dedicated database
php dock tenant:create

# List all registered tenants
php dock tenant:list

# Run migrations for all valid tenants
php dock tenant:migrate
```

### Features & Packages

#### Wallet & Payments

```bash
# Display wallet balance and statistics
php dock wallet:balance

# Reconcile wallet balance with transaction ledger
php dock wallet:reconcile

# Display wallet transaction history
php dock wallet:transaction

# Verify pending payments with gateways
php dock pay:verify-pending
```

#### Subscriptions & Campaigns (Wave & Blish)

```bash
# Send renewal reminders for subscriptions
php dock wave:remind

# Process renewals for expiring subscriptions
php dock wave:renew

# Process and send scheduled campaigns
php dock blish:process
```

#### Workflows & Tasks

```bash
# Create a new workflow class
php dock workflow:create Name Module

# Process and spawn recurring task instances
php dock flow:recur

# Process and send all due task reminders
php dock flow:remind

# Process due reminders and dispatch notifications
php dock hub:remind
```

#### Security & Permissions

```bash
# Sync roles and permissions from configuration
php dock permit:sync

# Clear and rebuild permission cache
php dock permit:cache

# Show status of all feature flags
php dock rollout:status
```

#### Storage & Vault

```bash
# Check storage usage for an account
php dock vault:usage

# Allocate storage quota to an account
php dock vault:allocate

# Create a backup of account storage
php dock vault:backup

# Wipe all storage for an account
php dock vault:wipe
```

### File & Directory Operations

```bash
# Create a file
php dock file:create path/to/file.php

# Delete a file
php dock file:delete path/to/file

# Create a directory
php dock directory:create path/to/directory

# Delete a directory
php dock directory:delete path/to/directory
```

## Examples

### Setting Up a New Module

```bash
# Create module
php dock module:create Blog

# Create model
php dock model:create Post Blog

# Create controller
php dock controller:create PostController Blog

# Create service
php dock service:create PostService Blog

# Create migration
php dock migration:create CreatePostsTable

# Create view model
php dock view:create-model PostViewModel Blog

# Create view templates
php dock view:create-template index Blog
php dock view:create-template show Blog
php dock view:create-template create Blog
```

### Database Workflow

```bash
# Create migration
php dock migration:create CreateUserTable

# Run migrations
php dock migration:run

# Check status
php dock migration:status

# Create seeder
php dock seeder:create UserSeeder

# Run seeders
php dock seeder:run
```

### Queue Workflow

```bash
# Create a task
php dock task:create SendEmailTask Notifications

# Start worker
php dock worker:start

# Check worker status
php dock worker:status

# Check queue
php dock queue:check

# Restart worker
php dock worker:restart
```

### Development Workflow

```bash
# Generate app key
php dock key:generate

# Start development server with worker
php dock dev

# In another terminal, run migrations
php dock migration:run

# Seed database
php dock seeder:run

# Check queue status
php dock queue:check
```

## Command Categories

| Category              | Commands                                | Purpose                        |
| --------------------- | --------------------------------------- | ------------------------------ |
| **Development**       | dev, playground, key:generate           | Development tools              |
| **Database**          | database:\*                             | Database management            |
| **Migration**         | migration:\*                            | Schema migrations              |
| **Seeder**            | seeder:\*                               | Data seeding                   |
| **Queue**             | queue:\*                                | Queue management               |
| **Worker**            | worker:\*                               | Worker daemon control          |
| **Cache**             | cache:flush                             | Cache management               |
| **Generators**        | \*:create                               | Code generation                |
| **Droppers**          | \*:delete                               | Code deletion                  |
| **Code Quality**      | inspect, sail                           | Analysis & Production Checks   |
| **System & Packages** | anchor:hydrate, anchor:update, download | Framework & Package management |
| **Files**             | file:_, directory:_                     | File operations                |

## Tips

- **Use --help**: Every command supports `--help` for detailed usage
- **Tab completion**: Enable shell completion with `php dock completion`
- **Verbose output**: Use `-v`, `-vv`, or `-vvv` for more details
- **Quiet mode**: Use `-q` to suppress output except errors
- **Non-interactive**: Use `-n` for scripts/automation

## Creating Custom Commands

Create a command class in `System/Cli/Commands/` or `App/Commands/`.

```php
namespace App\Commands;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class MyCustomCommand extends Command
{
    protected static $defaultName = 'my:command';
    protected static $defaultDescription = 'My custom command';

    protected function configure(): void
    {
         $this->setName('my:command')
              ->setDescription('My custom command');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln('Hello from custom command!');
        return Command::SUCCESS;
    }
}
```
