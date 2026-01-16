# Routine Helper

The `Helpers\Routine` namespace focuses on managing execution flows, specifically for sequential tasks and data pipelining.

## Pipe

The `Pipe` class allows you to execute a sequence of tasks where the output of one task becomes the input for the next. This is useful for data transformation pipelines.

#### start

```php
static start(mixed $initial = null): self
```

Initializes a new Pipe with a starting value.

- **Example**: `Pipe::start($user_input)`.

#### through

```php
through(callable $stage): self
```

Passes the current data through a transformation step.

- **Use Case**: Sanitizing, then slugging, then truncating a string in one sequence.
- **Example**: `->through('trim')->through('Str::slug')`.

#### then

```php
then(callable $finalCallback): self
```

Defines a callback to be executed on the final result before completion.

#### run

```php
run(): mixed
```

Trigger the execution of all registered stages and return the final computed value.

```php
use Helpers\Routine\Pipe;

$result = Pipe::start(' hello ')
    ->through('trim')
    ->through('strtoupper')
    ->through(fn($s) => $s . '!')
    ->run(); // "HELLO!"
```

## Sync

The `Sync` class manages the sequential execution of independent tasks. It allows for "before" and "after" hooks that run around each task.

#### new

`static new(): self`

Creates a new Sync instance.

#### before / after

`before(callable $callback): self`
`after(callable $callback): self`

Sets callbacks to run before and after each task. The `before` callback's return value is passed to the task, and the task's return value is passed to the `after` callback.

#### task

`task(callable $task): self`

Adds a task to the execution list.

#### execute

`execute(): array`

Runs all tasks in order and returns an array of results (post-processed by the `after` hook).

```php
use Helpers\Routine\Sync;

$results = Sync::new()
    ->before(fn() => date('Y-m-d H:i:s'))
    ->task(fn($time) => "Task 1 started at $time")
    ->task(fn($time) => "Task 2 started at $time")
    ->execute();
```

## Related

- [Lottery](lottery-helper.md) - Probabilistic execution routines
- [Benchmark](benchmark-helper.md) - Measuring execution time of routines
