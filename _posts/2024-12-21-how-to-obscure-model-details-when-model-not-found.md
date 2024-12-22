---
title: "Avoid Leaking Model Info: Securing Responses When a Model Is Not Found"
date: 2024-12-21
tags: [Laravel, Eloquent]
description: Don't want your model's class name and ID leaked to end-users? Read on
---

## Laravel's ModelNotFoundException
Laravel has some handy functionality to bail when a model can't be found.  It can happen in a number of places,
but the two places I most often see it get raised are in route model binding and calling `Model::findOrFail()`.

### Implicit Route Model Binding
[Implicit route model binding](https://laravel.com/docs/11.x/routing#implicit-binding) is a huge win
for Laravel developers. It lets us typehint a Model in our controller method and
have it injected when the route is requested. Here's an example.

Say we have a model called Subscription.
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Subscription extends Model
{
    protected $guarded = [];
}
```

And a route registered in our `routes/api.php` file.
```php
Route::get(
    'subscriptions/{subscription}',
    function (Subscription $subscription) {
        return response()->json(['ok' => true]);
    });
```

When we make a request like `curl -H "Accept: application/json" http://127.0.0.1:8000/api/subscriptions/1234`,
we will get a response like this:
```json
{
  "ok": true
}
```

And if the subscription isn't found by that ID, we'll receive a 404 with a response like this
```json
{
    "message": "No query results for model [App\\Models\\Subscription] 1234"
}
```

### Model *OrFail() methods
If you want to find a particular model by some condition and throw an exception if it doesn't exist,
Laravel has you covered. Take for instance a route like this:
```php
Route::post(
    'subscriptions/{subscription}/share/{userId}',
    function (Subscription $subscription, int $userId) {
        $userToShareWith = User::where('is_active', true)->findOrFail($userId);
        // ... create a new subscription for them
        return response()->json(['ok' => true], 201);
    });
```

If we make a request for a user who doesn't exist by that ID, we'll get an exception
```json
{
  "message": "No query results for model [App\\Models\\User] 200"
}
```

## The problem
Laravel's default model not found exception is a bit too revealing to our end-user.

```json
{
  "message": "No query results for model [App\\Models\\Subscription] 1234"
}
```

At a glance, the user can tell that a primary key value doesn't exist in your database, the namespace of your model,
and can reasonably infer you've built your application with Laravel. I see this as less than ideal,
and worry about this information becoming an attack vector.

If you have gone through the pain of obscuring all internal integer IDs by only revealing UUIDs in API responses
and routes, then it's very simple to accidentally expose them.

```php
Route::get(
    'users/{user:uuid}/company-details',
    function (User $user) {
        $company = Company::findOrFail($user->company_id);
        // ... retrieve all the data you need to build a json response
    });
```

Your request is `GET /users/db418725-ffdf-4273-be93-cd9b2bb00ca6/company-details`,
but if the company doesn't exist, you'll get a response like this.
```json
{
  "message": "No query results for model [App\\Models\\Company] 287"
}
```

## The solution
If you're using Laravel 11, then you can modify the `bootstrap/app.php` file to create a custom handler for the
`ModelNotFoundException` and modify its return value.

```php
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withExceptions(function (Exceptions $exceptions) {
        if (app()->isProduction()) {
            $exceptions->map(
                ModelNotFoundException::class,
                function (ModelNotFoundException $e): NotFoundHttpException {
                    return new NotFoundHttpException("Not found", $e);
                }
            );
        }
    })->create();
```

Using the above, we modify the Laravel exception handler to return a generic 404 response with a message like this:

```json
{
  "message": "Not found"
}
```

As an added bonus, we only enable this behavior on production. In our local development and staging environments,
we will let the normal response be returned to make debugging easier.

### A quick aside on route model binding
If your model uses the `HasUniqueStringIds` trait, or any of its descendents such as `HasUuids`, `HasUlids`,
`HasVersion7Uuids` you'll see that the `ModelNotFoundException` is raised if a route parameter doesn't
match the specified valid type.

For instance, if you're employing HasUuids and a user requests `GET /podcasts/this-is-not-a-uuid`,
`this-is-not-a-uuid` is not a UUID and returns a 404.  Though this throws the same exception as if the row could
not be found in the database, the database is never queried because the binding fails during the validation of the key.

## In Closing
Laravel offers a lot of great functionality for simplifying our lives through implicit route model binding and
the host of functions on the Eloquent builder which raise an exception if the row cannot be found in the database.
While the default behavior may be a bit too revealing about your application, architecture, and database, Laravel also
makes it easy to override this behavior. 
