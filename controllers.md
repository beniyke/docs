# Controllers

Controllers handle incoming HTTP requests and return responses. They act as the glue between routes, services, and views.

## Basic Controller

Controllers extend `App\Core\BaseController`:

### Using CLI

```bash
# Create a standard controller in the 'Account' module
php dock controller:create Profile Account

# Create an API controller in the 'Account' module
php dock controller:create User Account --api
```

### Manual Creation

```php
namespace App\Account\Controllers;

use App\Core\BaseController;
use Helpers\Http\Response;

class TweetController extends BaseController
{
    public function index(): Response
    {
        return $this->asView('tweets');
    }
}
```

## Service-First Approach

Anchor encourages a **service-first architecture**. Business logic should live in service classes, not controllers:

```php
namespace App\Auth\Controllers;

use App\Services\Auth\WebAuthService;
use App\Validations\Form\LoginFormRequestValidation;
use App\Auth\Views\Models\LoginViewModel;
use App\Core\BaseController;
use Helpers\Http\Response;

class LoginController extends BaseController
{
    public function index(LoginViewModel $login_view_model): Response
    {
        return $this->asView('login', compact('login_view_model'));
    }

    public function attempt(LoginFormRequestValidation $validator): Response
    {
        if (!$this->request->isPost()) {
            return $this->response->redirect($this->request->fullRoute());
        }

        $validator->validate($this->request->post());

        if ($validator->has_error()) {
            $this->flash->error($validator->errors());
            return $this->response->redirect($this->request->fullRoute());
        }

        // Delegate to service
        if (!$this->auth->login($validator->getRequest())) {
            return $this->response->redirect($this->request->fullRoute());
        }

        return $this->handleLoginRedirect();
    }

    private function handleLoginRedirect(): Response
    {
        $redirect_to = $this->request->fullRouteByName('home');
        return $this->response->redirect($redirect_to);
    }
}
```

## Dependency Injection

Type-hint dependencies in controller methods - the IoC container will resolve them automatically:

```php
public function index(UserService $userService, LoginViewModel $viewModel): Response
{
    $users = $userService->getAllActiveUsers();
    return $this->asView('users', compact('users', 'viewModel'));
}
```

### Common Dependencies

- **Services**: Business logic (e.g., `UserService`, `AuthService`)
- **ViewModels**: View-specific data preparation
- **Validators**: Form/request validation
- **Requests**: Custom request objects

## BaseController Features

All controllers extending `BaseController` have access to several core properties and methods for managing common tasks.

### Core Properties

#### $this->request

An instance of `Helpers\Http\Request`, providing access to input data, environment variables, and request state.

#### $this->response

An instance of `Helpers\Http\Response`, used to build and send HTTP responses.

#### $this->flash

An instance of `Helpers\Http\Flash`, for managing one-time session messages (feedback, errors).

#### $this->auth

The `AuthServiceInterface` instance, handling user authentication and authorization.

#### $this->viewData

Provides access to structured view models like `layout_view_model` and `user_view_model`.

### Core Methods

#### asView

```php
asView(string $template, array $data = [], ?callable $callback = null): Response
```

Renders a view template located in the controller's module directory. It automatically passes the `layout` view model to the view.

- **Use Case**: Displaying a user profile page or a list of items.
- **Example**: `return $this->asView('profile', ['user' => $user])`.

#### asJson

```php
asJson(array $data, int $code = 200): Response
```

Returns a JSON response with the appropriate `Content-Type` header.

- **Use Case**: Returning data for AJAX requests or mobile app APIs.
- **Example**: `return $this->asJson(['status' => 'success', 'id' => 1])`.

#### asApiResponse

```php
asApiResponse(bool $is_successful, string $message, mixed $data = null, int $status_code = 200): Response
```

Wraps data in a standardized API structure: `['status' => bool, 'message' => string, 'data' => mixed]`.

- **Use Case**: Ensuring consistent API responses across the entire application.

#### unauthorized

```php
unauthorized(): Response
```

A convenience method that returns a 401 Unauthorized JSON response.

## Returning Responses

### Rendering Views

Use `asView()` to render templates:

```php
// Simple view
return $this->asView('tweets/show');

// With data
return $this->asView('tweets/show', ['tweet' => $tweet]);

// With compact
return $this->asView('tweets/show', compact('tweet', 'comments'));
```

### JSON Responses

Use `asJson()` for API endpoints:

```php
return $this->asJson([
    'status' => 'success',
    'data' => $tweet
]);

// With status code
return $this->asJson(['error' => 'Not found'], 404);
```

### Redirects

```php
// Redirect to route
return $this->response->redirect($this->request->fullRoute('tweets'));

// Redirect to named route
return $this->response->redirect($this->request->fullRouteByName('home'));

// Redirect back
return $this->response->back();
```

## Business Logic & Services

To keep controllers "thin" and manageable, move complex business logic into **Services**. This makes your code reusable and easier to test.

### Define the Service

Create a class that encapsulates a specific domain logic (e.g., managing users).

```php
namespace App\Services;

use App\Models\User;
use Helpers\Encryption\Drivers\SymmetricEncryptor;

class UserService
{
   public function __construct(
       private readonly SymmetricEncryptor $encryptor
   ) {}

   public function confirmUser(array $credentials): ?User
   {
       $user = User::query()->where('email', $credentials['email'])->first();

       if (!$user || !$this->encryptor->verifyPassword($credentials['password'], $user->password)) {
           return null;
       }

       return $user;
   }
}
```

### Inject into Controller

Type-hint the service in your controller method. The framework will automatically resolve and inject it.

```php
// App/Auth/Controllers/LoginController.php
namespace App\Auth\Controllers;

use App\Services\UserService;
use App\Validations\Form\LoginFormRequestValidation;
use Helpers\Http\Response;

class LoginController extends BaseController
{
    public function authenticate(
        LoginFormRequestValidation $validator,
        UserService $userService
    ): Response {
        // Delegate logic to the service
        $user = $userService->confirmUser($validator->validated()->data());

   if (!$user) {
       $this->flash->error('Invalid credentials');
       return $this->response->redirect($this->request->fullRoute('auth/login'));
   }

   // Login successful...
}
```

## Repository Pattern (Optional)

While services are encouraged, you can optionally use the repository pattern for data access:

```php
namespace App\Repositories;

use App\Models\Tweet;

class TweetRepository
{
    public function all()
    {
        return Tweet::all();
    }

    public function findById(int $id): ?Tweet
    {
        return Tweet::find($id);
    }

    public function create(array $data): Tweet
    {
        return Tweet::create($data);
    }
}
```

> The framework encourages services over repositories. Use repositories only if your team prefers that pattern.

## Validation

Use validation classes to validate requests:

```php
namespace App\Auth\Validations\Form;

use App\Core\BaseRequestValidation;

class LoginFormRequestValidation extends BaseRequestValidation
{
    public function expected(): array
    {
        return ['email', 'password'];
    }

    public function rules(): array
    {
        return [
            'email' => [
                'type' => 'email',
                'is_valid' => ['email']
            ],
            'password' => [
                'type' => 'string'
            ]
        ];
    }

    public function parameters(): array
    {
        return [
            'email' => 'Email Address',
            'password' => 'Password',
        ];
    }
}
```

## Best Practices

- **Keep controllers thin**: Move business logic to services
- **Use dependency injection**: Type-hint dependencies in methods
- **Use services for business logic**: Not repositories
- **Validate early**: Use validation classes
- **Use flash messages**: Provide user feedback
- **Return proper responses**: Views, JSON, or redirects
- **Handle errors gracefully**: Check for failures and provide feedback
