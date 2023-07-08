---
title: "Set your Gate's default rejection response"
date: 2023-07-06
tags: Laravel, authorization
---
## Background
There are at least two ways to use a policy within your controller methods.

1. You can call `authorizeResource()` within the [Controller's constructor](https://laravel.com/docs/10.x/authorization#authorizing-resource-controllers).
2. You can use `Route::can()` in your [route definition](https://laravel.com/docs/10.x/authorization#via-middleware).

But maybe you don't like the fact that you don't have control over the response that's returned.

Say for instance you have a controller like this:

```php
// app/Http/Controllers/TransactionController.php

use Illuminate\Http\Request;
use App\Models\Transaction;

class TransactionController extends Controller
{
    public function show(Request $request, Transaction $transaction)
    {
        return $transaction;
    }
}
```

and a routes file like:
```php
// routes/api.php

Route::get('transaction/{transaction}', [App\Http\Controllers\TransactionController::class, 'show'])->can('view', 'transaction');
```

and finally, a policy like this:

```php
// app/Policies/TransactionPolicy.php

use App\Models\Transaction;
use App\Models\User;

class TransactionPolicy
{
    public function view(?User $user = null, Transaction $transaction): bool
    {
        return $transaction->user_id === $user?->id;
    }
}
```

This should make it so that only the user who created the transaction can view the API endpoint.  So, what if happens if an user requests a transaction they did not create? Let's say transaction ID 123 belongs to a different user. Requesting `/transaction/123` will throw a `AccessDeniedHttpException`, which returns a 403 response with the message "This action is unauthorized."

```zsh
$ curl "https://mess-around.test/api/transaction/123" -H "Accept: application/json" -v
< HTTP/2 403 
< server: nginx/1.25.1
< content-type: application/json
< x-powered-by: PHP/8.2.6
{
    "message": "This action is unauthorized."
}
```

But what happens if you request a transaction ID that doesn't exist? Now the response is different: you'll receive a 404 with "No query result found for \[App\\Models\\Transaction] 999".

```zsh
$ curl "https://mess-around.test/api/transaction/999999" -H "Accept: application/json" -v
< HTTP/2 404 
< server: nginx/1.25.1
< content-type: application/json
< x-powered-by: PHP/8.2.6
{
    "message": "No query results for model [App\\Models\\Transaction] 999"
}
```


## Problem
In some cases, this is helpful and totally fine. But what if you have a public API and wish to avoid exposing which models do or do not exist?  You wouldn't want someone to be able to see how many transactions your system has. Or maybe you have [scoped bindings](https://laravel.com/docs/10.x/routing#implicit-model-binding-scoping) and don't want to give a malicious actor the ability to see if a transaction belongs to a user.

## Solution
With Laravel's [10.14.0 release](https://github.com/laravel/framework/releases/tag/v10.14.0), you can customize the Gate's default denial response. This can be added to your AppServiceProvider.

```php

use Gate;
use Illuminate\Auth\Access\Response;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        Gate::setDenialResponse(Response::denyAsNotFound());
    }
}
```

Now, when you request a transaction that exists but the user is not permitted to see it, a 404 response will be returned.

```zsh
$ curl "https://mess-around.test/api/transaction/123" -H "Accept: application/json" -v
< HTTP/2 404 
< server: nginx/1.25.1
< content-type: application/json
< vary: Accept-Encoding
< x-powered-by: PHP/8.2.6
{
    "message": "Not Found"
}
```

Now in the example above, a transaction denied by the TransactionPolicy will return a 404, the same as if the model does not exist.

While the status code is the same, the response message belies whether the model exists or is not visible to the authenticated user.  If you want to make sure the messages are consistent so that both not found and not accessible return the same message, we'll need to modify `App\Exceptions\Handler` to change how a ModelNotFound exception is rendered.

```php
// app/Exceptions/Handler.php


namespace App\Exceptions;

use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

class Handler extends ExceptionHandler
{
    /**
     * The list of the inputs that are never flashed to the session on validation exceptions.
     *
     * @var array<int, string>
     */
    protected $dontFlash = [
        'current_password',
        'password',
        'password_confirmation',
    ];

    /**
     * Register the exception handling callbacks for the application.
     */
    public function register(): void
    {
        $this->map(ModelNotFoundException::class, fn () => new NotFoundHttpException('Not Found'));
    }
}
```

This makes it so that ModelNotFoundExceptions renders the same as a policy denial.

```zsh
$ curl "https://mess-around.test/api/transaction/999999" -H "Accept: application/json"
{
    "message": "Not Found"
}
```
