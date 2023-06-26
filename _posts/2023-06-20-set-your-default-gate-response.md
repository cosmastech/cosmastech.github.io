---
title: "Set your Gate's default rejection response"
date: 2023-06-20
tags: Laravel, authorization
---

There are at least two ways to use a policy within your controller methods.

1. You can call `authorizeResource()` within the [Controller's constructor](https://laravel.com/docs/10.x/authorization#authorizing-resource-controllers).
2. You can use `Route::can()` in your [route definition](https://laravel.com/docs/10.x/authorization#via-middleware).

But maybe you don't like the fact that you don't have control over the response that's returned.

Say for instance you have a controller like this:

```php
use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use App\Models\Transaction;

class TransactionsController extends Controller
{
    public function show(Request $request, Transaction $transaction)
    {
        return $transaction;
    }
}
```

and a routes file like:
```php
// api.php

Route::get('transaction/{transaction}', [App\Http\Controllers\PostController::class, 'show'])->can('view', 'transaction');
```

and finally, a policy like this:

```php
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

https://github.com/laravel/framework/pull/47436/
