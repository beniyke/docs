# Testing Guidelines

## Unified Directory Mirroring

The `tests/` directory must strictly mirror the framework source. Every directory is an isolated testing island.

### Structure

```
tests/
├── App/                          # Application-specific tests
│   ├── Unit/
│   ├── Feature/
│   ├── Integration/
│   ├── Architecture/
│   └── Fixtures/
├── System/                       # Framework core tests
│   ├── Unit/
│   │   ├── Cli/
│   │   ├── Core/
│   │   ├── Database/
│   │   ├── Helpers/
│   │   ├── Mail/
│   │   ├── Notify/
│   │   ├── Package/
│   │   ├── Queue/
│   │   ├── Security/
│   │   └── Testing/
│   ├── Feature/
│   ├── Integration/
│   ├── Architecture/
│   ├── Fixtures/                  # Test-specific models, DTOs
│   │   ├── Models/
│   │   ├── DTOs/
│   │   └── Factories/
│   └── Support/                   # Shared test utilities
├── Packages/                      # Package-specific tests
│   ├── {PackageName}/
│   │   ├── Unit/
│   │   ├── Feature/
│   │   ├── Integration/
│   │   └── Architecture/
│   └── ...
├── Pest.php                       # Global test configuration
├── TestCase.php                   # Base test case
└── PackageTestCase.php            # Package test case
```

### Test Type Definitions

| Directory       | Purpose                                           | Database      | Container |
| --------------- | ------------------------------------------------- | ------------- | --------- |
| `Unit/`         | Isolated logic testing. No external dependencies. | Never         | Never     |
| `Feature/`      | Request-to-Response journeys. Uses `TestCase`.    | Transactional | Yes       |
| `Integration/`  | External API contracts & ServiceProvider wiring.  | Transactional | Yes       |
| `Architecture/` | Structural rules enforcement via `arch()`.        | Never         | Never     |
| `Fixtures/`     | Test-specific models, DTOs, factories.            | N/A           | N/A       |
| `Support/`      | Custom traits, helpers, mock builders.            | N/A           | N/A       |

## Database & Migration Rules

### Strict Rules

> **No Inline Migrations:** Test files must **never** contain migration logic, schema definitions, or `$this->dock('migrate:run')` calls in the setup.

### System-Led Schema

Tests must run against the core **System Migrations**. The `RefreshDatabase` trait handles:

- Running all migrations once per test class
- Wrapping each test in a transaction
- Rolling back after each test for isolation

```php
// CORRECT: Use RefreshDatabase trait
beforeEach(function () {
    $this->refreshDatabase();
});

// WRONG: Inline schema creation
beforeEach(function () {
    Schema::create('user', function ($table) {
        $table->id();
        $table->dateTimestamps();
    });
});
```

### Test Fixtures

Use the `Tests\System\Fixtures\` namespace for test-only models:

```php
// tests/System/Fixtures/Models/TestUser.php
namespace Tests\System\Fixtures\Models;

use Database\BaseModel;

class TestUser extends BaseModel
{
    protected string $table = 'user';
    protected array $fillable = ['name', 'email', 'status'];
}
```

### No Inline Helper Classes

> Helper classes (e.g., testable subclasses, stubs) must NOT be defined inline in test files.

Place all helper classes in the appropriate `Support/` directory:

```
tests/
├── System/
│   └── Support/           # Helper classes for system tests
│       ├── Queue/
│       │   └── TestableWorker.php
│       └── Security/
│           └── TestFirewall.php
└── Packages/
    └── {PackageName}/
        └── Support/       # Helper classes for package tests
```

**Correct:**

```php
// tests/System/Support/Queue/TestableWorker.php
namespace Tests\System\Support\Queue;

use Queue\Worker;

class TestableWorker extends Worker
{
    // Expose protected methods for testing
}
```

```php
// tests/System/Unit/Queue/WorkerTest.php
use Tests\System\Support\Queue\TestableWorker;

test('worker stops on signal', function () {
    $worker = new TestableWorker(...);
});
```

**Wrong:**

```php
// WRONG: Inline helper class in test file
class TestableWorker extends Worker { /* ... */ }

test('worker stops on signal', function () {
    // ...
});
```

### Model Factories

Use `System\Testing\Factories\Factory` for seeding consistent test data:

```php
// tests/System/Fixtures/Factories/UserFactory.php
namespace Tests\System\Fixtures\Factories;

use Testing\Factories\Factory;
use Tests\System\Fixtures\Models\TestUser;

class UserFactory extends Factory
{
    protected string $model = TestUser::class;

    public function definition(): array
    {
        return [
            'name' => $this->faker->name(),
            'email' => $this->faker->unique()->email(),
            'status' => 'active',
        ];
    }
}
```

---

## Boundary Coverage & Mutation Resistance

### Mutation-Resistant Patterns

Every logical branch must include a `with()` dataset covering boundaries and edge cases:

```php
test('validates age correctly', function (mixed $age, bool $expected) {
    $result = $validator->isAdult($age);

    expect($result)->toBe($expected); // Strict comparison kills mutants
})->with([
    // Boundary values
    'exactly 18 (boundary)' => [18, true],
    'just under 18' => [17, false],
    'just over 18' => [19, true],

    // Zero and negative
    'zero' => [0, false],
    'negative' => [-1, false],

    // Edge cases
    'max integer' => [PHP_INT_MAX, true],
    'float edge' => [17.9999, false],

    // Type coercion
    'string number' => ['18', true],
    'empty string' => ['', false],

    // Null handling
    'null' => [null, false],
]);
```

### Mutation Killing Strategies

| Mutant Type                 | How to Kill                                                    |
| --------------------------- | -------------------------------------------------------------- |
| `>` → `>=`                  | Test exact boundary: `18` returns `true`, `17` returns `false` |
| `<` → `<=`                  | Test both sides of boundary                                    |
| `&&` → `\|\|`               | Test when one condition is true, other is false                |
| `true` → `false`            | Assert exact boolean returns                                   |
| `return $x` → `return null` | Always assert the return value                                 |
| `$x++` → `$x--`             | Assert state change direction                                  |
| String mutations            | Assert exact string content                                    |

### Required Dataset Coverage

```php
// Standard boundary dataset for numeric validation
$numericBoundaries = [
    'zero' => [0],
    'negative_one' => [-1],
    'positive_one' => [1],
    'min_int' => [PHP_INT_MIN],
    'max_int' => [PHP_INT_MAX],
    'float_zero' => [0.0],
    'tiny_float' => [0.0000001],
    'negative_float' => [-0.1],
];

// Standard dataset for string validation
$stringBoundaries = [
    'empty' => [''],
    'whitespace' => ['   '],
    'single_char' => ['a'],
    'unicode' => ['émoji 🎉'],
    'very_long' => [str_repeat('a', 10000)],
    'null_byte' => ["test\0test"],
    'newlines' => ["line1\nline2"],
];

// Standard dataset for null/type handling
$nullAndTypes = [
    'null' => [null],
    'false' => [false],
    'true' => [true],
    'empty_array' => [[]],
    'zero_string' => ['0'],
];
```

---

## Execution Logic & Edge Cases

### Strict Typing

```php
<?php

declare(strict_types=1);

// Use `use` statements only
use Database\Query\Builder;
use Database\ConnectionInterface;

// Never use FQCN in test bodies
$builder = new Builder(...);
```

### Time Travel

Never use `time()`, `date()`, or `Carbon::now()` without freezing. Use `DateTimeHelper` for consistency:

```php
use Helpers\DateTimeHelper;
use App\Models\Subscription;

beforeEach(function () {
    DateTimeHelper::setTestNow('2025-01-15 10:00:00');
});

afterEach(function () {
    DateTimeHelper::setTestNow(); // Reset
});

test('subscription expires correctly', function () {
    $subscription = Subscription::create([
        'expires_at' => DateTimeHelper::now()->addDays(30),
    ]);

    // Travel to expiration
    DateTimeHelper::setTestNow(DateTimeHelper::now()->addDays(31));

    expect($subscription->isExpired())->toBeTrue();
});
```

### Negative Testing

Assert that the system fails correctly:

```php
test('throws on invalid email', function () {
    expect(fn() => $validator->validate('not-an-email'))
        ->toThrow(ValidationException::class, 'Invalid email format');
});

test('returns null for missing record', function () {
    $result = User::find(999999);

    expect($result)->toBeNull();
});

test('handles database connection failure', function () {
    // Simulate connection failure
    $this->mockConnection->shouldReceive('query')
        ->andThrow(new ConnectionException('Connection refused'));

    expect(fn() => $repository->findAll())
        ->toThrow(RepositoryException::class);
});
```

## Architecture Guardrails

Each module **must** contain an `ArchTest.php` using Pest `arch()`.

### Required Architecture Rules

```php
<?php

declare(strict_types=1);

describe('Module Architecture Rules', function () {

    // =================================
    // Unit Test Isolation
    // =================================

    arch('Unit tests do not use database')
        ->expect('Tests\\System\\Unit')
        ->not->toUse([
            'assertDatabaseHas',
            'assertDatabaseMissing',
            'assertDatabaseCount',
            'assertModelExists',
            'assertModelMissing',
        ]);

    arch('Unit tests do not touch the container')
        ->expect('Tests\\System\\Unit')
        ->not->toUse(['resolve', 'container', 'app']);

    arch('Unit tests do not make HTTP requests')
        ->expect('Tests\\System\\Unit')
        ->not->toUse(['get', 'post', 'put', 'patch', 'delete', 'getJson', 'postJson']);

    // =================================
    // Feature Test Structure
    // =================================

    arch('Feature tests extend System TestCase')
        ->expect('Tests\\System\\Feature')
        ->toExtend('Tests\\TestCase');

    arch('Package Feature tests extend PackageTestCase')
        ->expect('Tests\\Packages\\*\\Feature')
        ->toExtend('Tests\\PackageTestCase');

    // =================================
    // Code Quality
    // =================================

    arch('No debugging functions in tests')
        ->expect('Tests')
        ->not->toUse(['dd', 'dump', 'var_dump', 'print_r', 'var_export', 'ray']);

    arch('No sleep in tests')
        ->expect('Tests')
        ->not->toUse(['sleep', 'usleep', 'time_nanosleep']);

    arch('All test files declare strict types')
        ->expect('Tests')
        ->toUseStrictTypes();

    // =================================
    // Naming Conventions
    // =================================

    arch('Test files have Test suffix')
        ->expect('Tests')
        ->toHaveSuffix('Test')
        ->ignoring([
            'Tests\\TestCase',
            'Tests\\PackageTestCase',
            'Tests\\System\\Fixtures',
            'Tests\\System\\Support',
        ]);
});
```

### Package-Specific Rules

```php
// tests/Packages/Wallet/Architecture/ArchTest.php

arch('Wallet tests do not depend on Wave')
    ->expect('Tests\\Packages\\Wallet')
    ->not->toUse('Wave');

arch('Wallet unit tests use WalletFake')
    ->expect('Tests\\Packages\\Wallet\\Unit')
    ->toUse('Wallet\\Testing\\WalletFake');
```

---

## System/Testing Toolkit Reference

### Available Traits

| Trait                         | Location           | Use In               | Purpose                                      |
| ----------------------------- | ------------------ | -------------------- | -------------------------------------------- |
| `RefreshDatabase`             | `Testing\Concerns` | Feature, Integration | Auto-migrate, wrap in transaction, rollback  |
| `InteractsWithDatabase`       | `Testing\Concerns` | Feature              | `assertDatabaseHas()`, `assertModelExists()` |
| `InteractsWithFakes`          | `Testing\Concerns` | Unit, Feature        | Bootstrap fakes for Mail, Queue, Events      |
| `InteractsWithTime`           | `Testing\Concerns` | All                  | Time freezing, `travelTo()`                  |
| `MakesHttpRequests`           | `Testing\Concerns` | Feature              | `$this->get()`, `$this->postJson()`          |
| `InteractsWithAuthentication` | `Testing\Concerns` | Feature              | `actingAs()`, authenticated state            |
| `InteractsWithConsole`        | `Testing\Concerns` | Integration          | Dock command testing                         |
| `InteractsWithPackages`       | `Testing\Concerns` | Integration          | Package loading, service providers           |

### Available Fakes

| Fake               | Purpose            | Key Methods                                             |
| ------------------ | ------------------ | ------------------------------------------------------- |
| `HttpFake`         | Mock outgoing HTTP | `fake()`, `assertSent()`, `assertNotSent()`             |
| `MailFake`         | Mock email sending | `assertSent()`, `assertQueued()`, `assertNothingSent()` |
| `QueueFake`        | Mock job dispatch  | `assertPushed()`, `assertPushedOn()`                    |
| `EventFake`        | Mock events        | `assertDispatched()`, `assertNotDispatched()`           |
| `CacheFake`        | Mock cache         | `assertHas()`, `assertMissing()`                        |
| `NotificationFake` | Mock notifications | `assertSentTo()`, `assertNotSentTo()`                   |
| `StorageFake`      | Mock file storage  | `assertExists()`, `assertMissing()`                     |

### Usage Examples

```php
use Testing\Fakes\MailFake;
use Testing\Fakes\QueueFake;
use Mail\Contracts\MailerInterface;
use Queue\Contracts\QueueInterface;
use App\Services\UserService;
use App\Notifications\Email\WelcomeEmail;

beforeEach(function () {
    $this->mail = new MailFake();
    $this->queue = new QueueFake();

    // Register fakes in container
    container()->bind(MailerInterface::class, fn() => $this->mail);
    container()->bind(QueueInterface::class, fn() => $this->queue);
});

test('sends welcome email on registration', function () {
    $user = UserService::register([
        'email' => 'test@example.com',
        'name' => 'Test User',
    ]);

    $this->mail->assertSent(WelcomeEmail::class, function ($mail) use ($user) {
        return $mail->to === $user->email;
    });
});
```

## HTTP Resiliency Testing

### Pattern for External API Testing

```php
use Testing\Fakes\HttpFake;

beforeEach(function () {
    $this->http = new HttpFake();
});

describe('External API Resilience', function () {

    test('handles 500 server error gracefully', function () {
        $this->http->fake([
            'api.external.com/*' => ['status' => 500, 'body' => 'Internal Server Error'],
        ]);

        expect(fn() => $this->service->fetchData())
            ->toThrow(ExternalApiException::class, 'Server error');
    });

    test('handles 401 unauthorized', function () {
        $this->http->fake([
            'api.external.com/*' => ['status' => 401, 'body' => '{"error": "Unauthorized"}'],
        ]);

        expect(fn() => $this->service->fetchData())
            ->toThrow(AuthenticationException::class);
    });

    test('handles 429 rate limiting with retry', function () {
        $attempts = 0;

        $this->http->fake([
            'api.external.com/*' => function () use (&$attempts) {
                return ++$attempts < 3
                    ? ['status' => 429, 'body' => '', 'headers' => ['Retry-After' => '1']]
                    : ['status' => 200, 'body' => '{"success": true}'];
            },
        ]);

        $result = $this->service->fetchData();

        expect($result->success)->toBeTrue();
        $this->http->assertSentCount(3);
    });

    test('handles network timeout', function () {
        $this->http->fake([
            'api.external.com/*' => ['status' => 504, 'body' => ''],
        ]);

        expect(fn() => $this->service->fetchData())
            ->toThrow(TimeoutException::class);
    });

    test('handles malformed JSON response', function () {
        $this->http->fake([
            'api.external.com/*' => ['status' => 200, 'body' => 'not valid json {'],
        ]);

        expect(fn() => $this->service->fetchData())
            ->toThrow(MalformedResponseException::class);
    });
});
```

## Test Contracts & Expectations

Every test file must adhere to these contracts:

### File-Level Requirements

- **Declare Strict Types:** `declare(strict_types=1);` at file top
- **Use Describe Blocks:** Group related tests with `describe()` for clarity
- **Self-Documenting Names:** Test names must describe behavior, not implementation

### Test-Level Requirements

```php
// GOOD: Behavior-focused name
test('returns zero when cart is empty', function () { ... });

// BAD: Implementation-focused name
test('getTotal method with no items', function () { ... });
```

### State Management

```php
beforeEach(function () {
    // Setup fresh state for each test
    $this->sut = new SystemUnderTest();
});

afterEach(function () {
    // Clean up any side effects
    Mockery::close();
    Carbon::setTestNow();
});
```

### Deterministic Data

```php
// WRONG: Non-deterministic
test('creates user with random email', function () {
    $user = User::create(['email' => 'user' . rand() . '@test.com']);
});

// CORRECT: Deterministic with faker seed or fixed data
test('creates user with email', function () {
    $user = User::create(['email' => 'test@example.com']);
    expect($user->email)->toBe('test@example.com');
});
```

### Isolation

Each test must be runnable in isolation:

```bash
# This must always work
./vendor/bin/pest --filter="returns zero when cart is empty"
```

## Mutation Testing Configuration

### Pest Configuration

```php
// tests/Pest.php

uses()->group('mutation')->in('System/Unit');
uses()->group('mutation')->in('Packages/*/Unit');
```

### Running Mutation Tests

```bash
# Run mutation testing
./vendor/bin/pest --mutate

# With minimum score requirement
./vendor/bin/pest --mutate --min=100

# For specific directory
./vendor/bin/pest --mutate tests/System/Unit/Helpers
```

### Mutation Score Targets

| Test Type   | Minimum MSI | Notes                                    |
| ----------- | ----------- | ---------------------------------------- |
| Unit        | **100%**    | No surviving mutants allowed             |
| Feature     | **90%**     | Some integration complexity accepted     |
| Integration | **85%**     | External dependencies may limit coverage |

### Excluded from Mutation

Add to `pest.php`:

```php
// Files excluded from mutation testing
uses()->group('no-mutation')->in([
    'System/Fixtures',
    'System/Support',
    'System/Architecture',
    'Packages/*/Fixtures',
]);
```

## Code Coverage Requirements

### Targets

| Metric          | Target | Critical Paths |
| --------------- | ------ | -------------- |
| Line Coverage   | ≥ 90%  | ≥ 95%          |
| Branch Coverage | ≥ 85%  | ≥ 90%          |
| Path Coverage   | ≥ 80%  | ≥ 85%          |

### Running Coverage

```bash
# Generate coverage report
composer test:coverage

# With HTML output
./vendor/bin/pest --coverage --coverage-html=coverage-report
```

### PHPUnit Configuration

```xml
<!-- phpunit.xml additions -->
<source>
    <include>
        <directory suffix=".php">App</directory>
        <directory suffix=".php">System</directory>
        <directory suffix=".php">packages</directory>
    </include>
    <exclude>
        <directory>System/Testing</directory>
        <directory>*/Migrations/*</directory>
        <file>*/Facades/*.php</file>
    </exclude>
</source>
```

## Parallel Testing Guidelines

Tests must be safe for parallel execution.

### Parallel Safety Rules

- **Database Isolation:** Each process uses separate transaction
- **File System:** Use unique temp directories per process
- **Singletons:** Never rely on static state between tests
- **Environment:** Never modify `$_ENV` or `$_SERVER`

### File System Isolation

```php
test('writes file correctly', function () {
    // Use unique temp path per test/process
    $tempPath = sys_get_temp_dir() . '/test_' . uniqid() . '_' . getmypid();

    try {
        $writer->write($tempPath, 'content');
        expect(file_get_contents($tempPath))->toBe('content');
    } finally {
        @unlink($tempPath);
    }
});
```

### Running Parallel Tests

```bash
# Run tests in parallel (8 processes by default)
./vendor/bin/pest --parallel

# With specific process count
./vendor/bin/pest --parallel --processes=4
```

## Test File Templates

### Unit Test Template

```php
<?php

declare(strict_types=1);

use App\Services\Calculator;
use App\Exceptions\DivisionByZeroException;

beforeEach(function () {
    $this->calculator = new Calculator();
});

describe('add', function () {
    test('adds two positive numbers', function () {
        expect($this->calculator->add(2, 3))->toBe(5);
    });

    test('adds negative numbers', function () {
        expect($this->calculator->add(-2, -3))->toBe(-5);
    });

    test('handles zero', function () {
        expect($this->calculator->add(5, 0))->toBe(5);
    });
})->covers(Calculator::class);

describe('divide', function () {
    test('divides two numbers', function () {
        expect($this->calculator->divide(10, 2))->toBe(5.0);
    });

    test('throws on division by zero', function () {
        expect(fn() => $this->calculator->divide(10, 0))
            ->toThrow(DivisionByZeroException::class);
    });
})->covers(Calculator::class);
```

### Feature Test Template

```php
<?php

declare(strict_types=1);

use Tests\System\Fixtures\Models\TestUser;

beforeEach(function () {
    $this->refreshDatabase();
});

describe('User Registration', function () {
    test('creates user with valid data', function () {
        $response = $this->postJson('/api/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'SecurePass123!',
        ]);

        $response->assertStatus(201);

        $this->assertDatabaseHas('user', [
            'email' => 'john@example.com',
        ]);
    });

    test('rejects duplicate email', function () {
        TestUser::create([
            'name' => 'Existing',
            'email' => 'john@example.com',
        ]);

        $response = $this->postJson('/api/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'SecurePass123!',
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['email']);
    });
});
```

### Integration Test Template

```php
<?php

declare(strict_types=1);

use Pay\PayManager;
use Pay\Drivers\StripeDriver;
use Testing\Fakes\HttpFake;

beforeEach(function () {
    $this->http = new HttpFake();
    $this->payManager = new PayManager($this->http);
});

describe('Stripe Integration', function () {
    test('processes payment successfully', function () {
        $this->http->fake([
            'api.stripe.com/charges' => [
                'status' => 200,
                'body' => json_encode(['id' => 'ch_123', 'status' => 'succeeded']),
            ],
        ]);

        $result = $this->payManager
            ->driver('stripe')
            ->charge(1000, 'USD');

        expect($result->isSuccessful())->toBeTrue();
        expect($result->getTransactionId())->toBe('ch_123');

        $this->http->assertSentWithMethod('POST', 'api.stripe.com/charges');
    });

    test('handles payment failure', function () {
        $this->http->fake([
            'api.stripe.com/charges' => [
                'status' => 402,
                'body' => json_encode(['error' => ['message' => 'Card declined']]),
            ],
        ]);

        $result = $this->payManager
            ->driver('stripe')
            ->charge(1000, 'USD');

        expect($result->isSuccessful())->toBeFalse();
        expect($result->getErrorMessage())->toBe('Card declined');
    });
});
```

### Architecture Test Template

```php
<?php

declare(strict_types=1);

describe('Module Architecture', function () {

    arch('uses strict types')
        ->expect('App\\Module')
        ->toUseStrictTypes();

    arch('does not use deprecated functions')
        ->expect('App\\Module')
        ->not->toUse(['mysql_query', 'ereg', 'split']);

    arch('services are final')
        ->expect('App\\Module\\Services')
        ->toBeFinal();

    arch('repositories implement interface')
        ->expect('App\\Module\\Repositories')
        ->toImplement('App\\Module\\Contracts\\RepositoryInterface');

    arch('no direct database in controllers')
        ->expect('App\\Module\\Controllers')
        ->not->toUse(['Database\\DB', 'Database\\Query\\Builder']);
});
```

## Output Specification

When creating tests, always provide:

### 1. Visual File Tree

```
tests/System/Unit/Helpers/
├── ValidationTest.php          # 15 tests, 100% MSI
├── StringTest.php              # 23 tests, 100% MSI
├── NumberTest.php              # 18 tests, 100% MSI
└── DateTimeHelperTest.php      # 12 tests, 100% MSI

tests/System/Feature/
├── DatabaseFeatureTest.php     # 8 tests, 92% MSI
└── QueueFeatureTest.php        # 6 tests, 90% MSI
```

### 2. Complete Runnable Code

All test code must be:

- Syntactically valid PHP 8.1+
- Immediately runnable with `./vendor/bin/pest`
- Free of placeholder comments like `// TODO` or `// implement`

### 3. Strategic Comments

Comments should explain **why**, not **what**:

```php
test('rejects age exactly at boundary', function () {
    // Boundary test: 17 is the last invalid age
    // This kills the >= mutant that would change 18 to 17
    expect($validator->isAdult(17))->toBeFalse();
});

test('accepts age exactly at boundary', function () {
    // Boundary test: 18 is the first valid age
    // Tests exact boundary to kill off-by-one mutants
    expect($validator->isAdult(18))->toBeTrue();
});
```

## Quick Reference Card

```bash
# Run all tests
./vendor/bin/pest

# Run specific suite
./vendor/bin/pest --testsuite=System

# Run with coverage
./vendor/bin/pest --coverage

# Run mutation testing
./vendor/bin/pest --mutate --min=100

# Run in parallel
./vendor/bin/pest --parallel

# Run single test
./vendor/bin/pest --filter="creates user with valid data"

# Run architecture tests only
./vendor/bin/pest tests/System/Architecture
```

> If a mutant survives, your test isn't testing what you think it's testing. Add a boundary assertion that would fail if the mutant's change was applied.
