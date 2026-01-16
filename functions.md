# Functions Reference

Anchor includes a variety of global "helper" PHP functions. Many of these functions are used by the framework itself; however, you are free to use them in your own applications if you find them convenient.

## Debug & Logging

#### dd

```php
dd(...$vars): void
```

Dumps the given variables to the browser and stops script execution immediately.

- **Use Case**: Quick debugging to inspect state at a specific point in the lifecycle.

#### dump

```php
dump(...$vars): void
```

Dumps variables to the browser without stopping execution.

```php
dump($users);
dump($user, $posts); // Multiple variables
```

#### logger

```php
logger(string $file): FileLogger
```

Retrieves a logger instance targeting a specific file in `App/storage/logs/`.

- **Example**: `logger('debug.log')->info('Task started');`.

#### benchmark

```php
benchmark(string $key, ?callable $callback = null): mixed
```

Measures the execution time of a specific block of code.

- **Example**: `$data = benchmark('api_call', fn() => curl()->get(...)->send());`.

## Container & Configuration

#### container

```php
container(): object
```

Returns the singleton instance of the IoC container.

#### auth

```php
auth(): AuthServiceInterface
```

Returns the authentication service instance.

#### resolve

```php
resolve(string $namespace): object
```

A shortcut to resolve a service from the container by its class name or interface.

- **Example**: `$auth = resolve(AuthServiceInterface::class);`.

### Config

`config(string $key)`

Get the specified configuration value.

```php
$appName = config('app.name');
$debug = config('app.debug');
```

### Env

`env(string $key, mixed $default = null)`

Gets the value of an environment variable.

```php
$debug = env('APP_DEBUG', false);
$dbHost = env('DB_HOST', 'localhost');
```

## HTTP & Requests

#### request

```php
request(): Request
```

Returns the current request instance.

- **Example**: `request()->ip();`.

#### response

```php
response(string $data = '', int $status = 200, array $headers = []): Response
```

Creates a new response object.

- **Example**: `return response('Not Found', 404);`.

#### redirect

```php
redirect(string $url = '', array $params = [], bool $internal = true): Response
```

Creates a redirect response.

- **Example**: `return redirect(url('home'));`.

#### url

```php
url(?string $path = null, array $query = []): string
```

Generates a fully qualified URL to the application root or a specific path.

- **Example**: `url('profile', ['user' => 1]);` -> `https://example.com/profile?user=1`.

### Route

`route(?string $uri = null, bool $re_route = false)`

Get or generate a route.

```php
$currentRoute = route();
$userRoute = route('user/profile');
```

### Curent Url

`current_url()`

Get the current URL.

```php
$url = current_url();
```

### Request Uri

`request_uri()`

Get the current request URI.

```php
$uri = request_uri();
```

### Agent

`agent()`

Get a user agent helper instance.

```php
$browser = agent()->browser();
$platform = agent()->platform();
```

### Curl

`curl()`

Get a cURL client instance.

```php
$response = curl()
    ->get('https://api.example.com/users')
    ->send();
```

## Session & Flash Messages

#### session

```php
session(?string $key = null, mixed $value = null): mixed
```

Retrieves or sets a session value. Returns the session manager if no key is provided.

- **Example**: `session('user_id', 1);`.

#### flash

```php
flash(?string $type = null, mixed $message = null): mixed
```

Retrieves or sets a flash message (one-time session message).

- **Example**: `flash('success', 'Saved!');`.

## Security

### Csrf Token

`csrf_token()`

Get the current CSRF token.

```php
$token = csrf_token();
```

#### encrypt / decrypt

```php
encrypt(mixed $value): string
decrypt(string $value): mixed
```

Helpers for quick symmetric encryption.

#### enc

```php
enc(string $driver = 'string'): Encryptor
```

Returns an encrypter instance for advanced operations like password hashing.

- **Example**: `enc()->hashPassword('secret');`.

## Files & Storage

#### filesystem

```php
filesystem(): FileSystem
```

Returns a filesystem helper instance for file operations.

#### mimes

```php
mimes(): Mimes
```

Returns a MIME type helper for guessing file types and extensions.

#### cache

```php
cache(string $path): Cache
```

Returns a cache instance for a specific key or namespace.

#### image

```php
image(?string $image_path = null): Image
```

Returns an image processing helper.

#### validate_upload

```php
validate_upload(array $file, array $options): bool
```

Checks if an uploaded file meets specific criteria (type, size).

#### upload_image / upload_document / upload_archive

```php
upload_image(array $file, string $destination, int $maxSize = 5242880): string
upload_document(array $file, string $destination, int $maxSize = 10485760): string
upload_archive(array $file, string $destination, int $maxSize = 52428800): string
```

Securely validates and moves uploaded files to a destination.

## Views & Assets

#### assets

```php
assets(string $file): string
```

Generates a URL for a public asset with automatic cache-busting.

- **Example**: `<link rel="stylesheet" href="<?= assets('css/main.css') ?>">`.

See [Assets Helper](html-helpers.md#assets) for more details on the underlying class and cache-busting behavior.

#### component / html

```php
component(string $name): Component
html(): HtmlBuilder
```

Helpers for generating HTML components and strings fluently.

## Arrays

#### arr

```php
arr(mixed $collection = null): ArrayCollection|Collections
```

Creates a collection object for fluent array manipulation.

- **Example**: `arr([1, 2])->map(fn($n) => $n * 2)->all();`.

## Strings

#### str / text

```php
str(mixed $string = null): Str|StrCollection
text(mixed $text = null): Text|TextCollection
```

Helpers for fluent string and text manipulation.

- **Example**: `str('hello world')->slug();` -> `"hello-world"`.

#### plural / inflect

```php
plural(string $value): string
inflect(string $value, int $count): string
```

Helpers for English word inflection.

- **Example**: `plural('child');` -> `"children"`.

## Date & Time

#### datetime

```php
datetime(mixed $date = null): DateTimeHelper
```

Returns a date-time helper for parsing, formatting, and manipulating dates.

- **Example**: `datetime('tomorrow')->format('Y-m-d');`.

## Money & Formatting

#### money

```php
money(int|float $amount, string $currency = 'USD'): Money
```

Creates a Money object for handling currency values securely.

#### money_parse / money_format

```php
money_parse(string $money, ?string $currency = null): Money
money_format(Money $money, ?string $locale = null): string
```

Helpers for parsing and formatting money values.

#### currency

```php
currency(string $code): Currency
```

Returns a Currency instance for a specific currency code.

#### number

```php
number(?int $num = null): Number
```

Returns a number helper for formatting and conversions.

- **Example**: `number(1234)->format();` -> `"1,234"`.

### Format

`format(mixed $data = null)`

Get a Format helper instance.

```php
$format = format($data);
```

## Mail & Notifications

#### mailer

```php
mailer(Mailable $mail): bool
```

Dispatches an email.

#### notify

```php
notify(string $channel): NotificationBuilder
```

Starts a notification flow for a specific channel (email, sms, etc.).

## CLI & Execution

#### dock

```php
dock(?string $command = null): CommandRunner
```

Executes CLI commands programmatically.

#### queue / job

```php
queue(string $namespace, mixed $payload, string $identifier = 'default'): QueuedJob
job(): QueueDispatcherInterface
```

Helpers for background job processing.

#### defer / defer_as

```php
defer(mixed $callbacks): void
defer_as(string $name, callable ...$callbacks): void
```

Schedules tasks to run after the response has been sent to the user.

### Deferrer

`deferrer()`

Get the deferrer instance.

```php
$deferrer = deferrer();
```

## Routing

#### route_name

```php
route_name(string $name): string
```

Retrieves a route path by its defined name.

## Miscellaneous

#### data / validator

```php
data(array $payload): Data
validator(): Validator
```

Helpers for creating data objects and validation instances.

## Miscellaneous

### Wave

`wave()`

Get the Wave manager service instance for handling subscriptions.

```php
$wave = wave();
```

### Null If Blank

`null_if_blank(mixed $value)`

Return null if the value is blank.

```php
$val = null_if_blank(''); // null
$val = null_if_blank('hello'); // "hello"
$val = null_if_blank(0); // 0 (not null)
```
