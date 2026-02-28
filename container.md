# Service Container

The Service Container, also known as the IoC (Inversion of Control) Container, is a powerful tool for managing class dependencies and performing dependency injection.

## Accessing the Container

The container is a singleton accessible via:

```php
use Core\Ioc\Container;

$container = Container::getInstance();
```

Or use the `container()` helper:

```php
$container = container();
```

## Binding Methods

#### bind

```php
bind(string $abstract, $concrete = null, bool $shared = false): void
```

Registers a basic binding in the container. A new instance is created every time the service is resolved unless `$shared` is set to `true`.

- **Use Case**: Mapping an interface to a specific implementation that doesn't need to be shared.
- **Example**: `$container->bind(MailerInterface::class, SmtpMailer::class);`.

#### singleton

```php
singleton(string $abstract, $concrete = null): void
```

Binds a service as a "shared" instance. Once resolved, the same instance is returned on all subsequent calls.

- **Use Case**: Database connections, configuration services, or session managers.
- **Example**: `$container->singleton(ConfigService::class);`.

#### instance

```php
instance(string $abstract, object $instance): void
```

Binds an existing object instance into the container.

- **Use Case**: Injecting a mock object during testing or a pre-configured service.

## Resolution Methods

#### get

```php
get(string $id): mixed
```

Resolves the given abstract type from the container. Throws an exception if the type cannot be built.

- **Use Case**: Manually retrieving a service in a non-injected context (like a legacy helper).

#### make

```php
make(string $abstract, array $parameters = []): object
```

Creates a new instance of the class, allowing you to pass specific constructor parameters while the container resolves the rest.

- **Use Case**: Spawning a dynamic service that requires some runtime data along with its dependencies.

## Resolving

### Get Method

```php
$mailer = $container->get(MailerInterface::class);
```

### Resolve Helper

```php
$mailer = resolve(MailerInterface::class);
```

### Make Method

Resolve with parameters:

```php
$service = $container->make(MyService::class, [
    'param1' => 'value1',
    'param2' => 'value2'
]);
```

## Optional Dependency Resolution

The container can gracefully handle dependencies on interfaces that are not bound if they are marked as optional (nullable or have a default value):

```php
public function __construct(
    private readonly ?TokenManagerInterface $tokens = null
) {
    // If TokenManagerInterface is not bound, $tokens will be null
}
```

This ensures the framework remains robust even when optional packages or bridges are not installed.

The container automatically resolves constructor dependencies:

```php
namespace App\Tweets\Controllers;

use App\Tweets\Services\TweetService;
use Mail\Mailer;
use Helpers\Http\Request;

class TweetController
{
    public function __construct(
        private readonly TweetService $tweetService,
        private readonly Mailer $mailer,
        private readonly Request $request
    ) {}

    public function index()
    {
        // Dependencies are automatically injected
        $tweets = $this->tweetService->getAll();
        return $this->asView('index', compact('tweets'));
    }
}
```

## Method Injection

The container can also inject dependencies into methods:

```php
public function store(
    TweetService $tweetService,
    LoginFormRequestValidation $validator
) {
    // Both parameters are automatically resolved
    $validator->validate($this->request->post());

    if (!$validator->has_error()) {
        $tweetService->create($validator->getData());
    }
}
```

## Contextual Binding

Bind different implementations based on context:

```php
$container->when(TweetController::class)
    ->needs(MailerInterface::class)
    ->give(SmtpMailer::class);

$container->when(UserController::class)
    ->needs(MailerInterface::class)
    ->give(SendgridMailer::class);
```

## Tagging

Tag multiple bindings and resolve them as a group:

```php
// Tag services
$container->tag([
    TweetService::class,
    UserService::class,
    PostService::class
], 'services');

// Resolve all tagged services
$services = $container->tagged('services');
```

## Calling Methods

Call a method with automatic dependency injection:

```php
$result = $container->call([MyClass::class, 'methodName'], [
    'param1' => 'value1'
]);
```

## Property Injection

The container supports property injection for public properties with type hints:

```php
class MyService
{
    public ConfigServiceInterface $config;
    public Mailer $mailer;

    public function doSomething()
    {
        // Properties are automatically injected
        $appName = $this->config->get('app.name');
    }
}
```

Disable property injection:

```php
$container->disablePropertyInjection();
```

## Convention-Based Binding

Set up automatic interface-to-implementation binding:

```php
$container->setConventionRules([
    'interface_namespace' => 'App\\Contracts\\',
    'implementation_namespace' => 'App\\Services\\',
    'interface_suffix' => 'Interface'
]);

// Now resolving App\Contracts\TweetInterface
// automatically binds to App\Services\Tweet
```

## Service Providers

Register service providers for deferred loading:

```php
$container->registerDeferredProvider(
    TweetServiceProvider::class,
    [TweetService::class, TweetRepository::class]
);
```

## Checking Bindings

```php
if ($container->has(MailerInterface::class)) {
    // Binding exists
}
```

## Real-World Examples

### Binding Services

```php
// In a service provider or bootstrap file
$container->singleton(ConfigServiceInterface::class, function() {
    return new ConfigService();
});

$container->bind(MailerInterface::class, function($container) {
    $config = $container->get(ConfigServiceInterface::class);
    return new SmtpMailer($config);
});

$container->bind(CacheInterface::class, function() {
    return new FileCache(storage_path('cache'));
});
```

### Using in Controllers

```php
namespace App\Account\Controllers;

use App\Services\UserService;
use App\Core\BaseController;
use Helpers\Http\Response;

class ProfileController extends BaseController
{
    public function __construct(
        private readonly UserService $userService
    ) {
        // UserService is automatically injected
    }

    public function show(): Response
    {
        $user = $this->userService->getCurrentUser();
        return $this->asView('profile', compact('user'));
    }
}
```

### Using in Services

```php
namespace App\Account\Services;

use Core\Services\ConfigServiceInterface;
use Helpers\Encryption\Drivers\SymmetricEncryptor;
use App\Models\User;

class UserService
{
    public function __construct(
        private readonly ConfigServiceInterface $config,
        private readonly SymmetricEncryptor $encryptor
    ) {}

    public function createUser(array $data): User
    {
        $data['password'] = $this->encryptor->hashPassword($data['password']);
        return User::create($data);
    }
}
```

### Resolving Manually

```php
// In any class
$mailer = resolve(Mail\Mailer::class);
$config = resolve(Core\Services\ConfigServiceInterface::class);

// Or
$mailer = container()->get(Mail\Mailer::class);
```

## Helper Functions

**container()**

Get the container instance:

```php
$container = container();
```

**resolve()**

Resolve a class from the container:

```php
$service = resolve(MyService::class);
```

## Best Practices

- **Type-hint dependencies**: Always use type hints for automatic injection

- **Use interfaces**: Bind to interfaces, not concrete classes
- **Singleton for shared state**: Use singleton for services that maintain state
- **Avoid service locator**: Don't pass the container around; use dependency injection
- **Use service providers**: Organize bindings in service providers
- **Constructor injection**: Prefer constructor injection over property injection
- **Keep constructors simple**: Don't do work in constructors

## Container Interface

The container implements the framework's `Core\Ioc\ContainerInterface`:

```php
use Core\Ioc\ContainerInterface;

function myFunction(ContainerInterface $container)
{
    if ($container->has(MailerInterface::class)) {
        $mailer = $container->get(MailerInterface::class);
    }
}
```

The interface provides the following core methods:

- `get(string $id)` - Resolve and return a service instance
- `has(string $id)` - Check if a binding exists
- `bind()`, `singleton()`, `instance()` - Register bindings
- `make()`, `call()` - Advanced resolution methods
- `when()`, `tag()`, `tagged()` - Contextual binding and tagging

> While the container's `get()` and `has()` methods have similar signatures to PSR-11, the framework uses its own container interface with additional features like contextual binding, tagging, and property injection.

## Advanced Features

### Resolved Instance Tracking

Get the container instance itself (as `ContainerInterface`):

```php
$instance = $container->getResolvedInstance();
```

### Reflection Caching

The container caches reflection data for better performance. Reflection is used to analyze constructor parameters and automatically resolve dependencies.

### Circular Dependency Detection

The container detects and prevents circular dependencies, throwing an exception if detected.
