---
title: "Cleaner middleware: static factory functions"
date: 2025-10-01
tags: [Laravel, Http, middleware, PHP]
description: A pattern to simplify parameterized middleware
---

[Route Middleware](https://laravel.com/docs/12.x/middleware) is a way to execute code before a request is handled by a controller.
Some examples of middleware in apps that I have built or used: complex authentication logic 😢, loading and validating a child-relationship,
setting contextual data, modifying headers, and caching responses which require heavy computation to produce.

Sometimes we want to be able to define a route's middleware with some kind of property.

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Container\Attributes\Singleton;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

#[Singleton]
class EnsureUserHasRoleMiddleware
{
    public function handle(Request $request, Closure $next, string $role): mixed
    {
        if (! $request->user()?->hasRole($role)) {
            throw new AuthorizationException();
        }

        return $next($request);
    }
}
```

In the above example, we are verifying that a user has a particular role. If they do not, then we throw an authorization exception.

When we define a route which uses this middleware, we must tell it which role to use.

```php
use App\Http\AdminController;
use App\Http\Middleware\EnsureUserHasRoleMiddleware;

Route::post('admin', [AdminController::class, 'create'])->middleware(EnsureUserHasRoleMiddleware::class . ':super-admin');
```

This instructs Laravel that before entering into the `POST /admin` route, pass the request through our middleware to ensure they
have the role of `'super-admin'`.

## Let's Clean It Up
As our application grows, we may end up repeating this same class-string + role again and again. It's not the nicest looking thing
in the world and it has the problems of relying on a magic string. In this case, I like to add a static method to my middleware
for building the route definition.

```php
class EnsureUserHasRoleMiddleware
{
    public static function forSuperAdmin(): string
    {
        return self::class . ':super-admin';
    }

    public static function forAdmin(): string
    {
        return self::class . ':admin';
    }

    // ...
}
```

Then we can tidy up our route definition like this:

```php
Route::post('admin', [AdminController::class, 'create'])->middleware(EnsureUserHasRoleMiddleware::forSuperAdmin());
```

If we are disciplined about using these static factory methods when defining middleware for our routes, our IDE can help us easily jump to
usages of these functions to see which routes require these roles.

---

This isn't ground-breaking, just something I feel makes for a cleaner codebase.

👋 Happy coding!
