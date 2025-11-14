---
title: "Improving unit test runtime in Laravel modular monolith"
date: 2025-11-13
tags: Laravel, Testing, modular, PHPUnit
---

I recently started a new job and was given my first exposure to a [modular monolith](https://laracasts.com/series/modular-laravel). On the surface, they have a lot
to love: all aspects of your app in one repo, one set of dependencies to keep updated, reduced code duplication, and allowing multiple engineering teams to work in
in their own isolated sections over the code are first that spring to mind. Modular monoalso of course present some challenges, and the one that kind of struck me
by surprise was test suite run time.

In previous projects, test suites usually took at most two minutes to run locally. Parallelization obviously helps a great deal, but it is not a silver bullet. I
have spent a great deal of my focus on writing tests that don't execute extraneous database queries:

* don't build more than you need
* leverage bulk inserts versus many inserts in a loop
* be mindful of model events that may trigger database queries
* test a single unit of code when possible
* leverage `Factory::make()` if writing to the database isn't strictly necessary

These are helpful guidelines, but an unfamiliar codebase with 8000+ tests is not something that can be optimized quickly, if those opportunities even exist.

## Wait, it takes how long?
In our CI pipeline, just the PHPUnit tests were taking between 8 and 10 minutes to execute fully. Heaven help you if you needed something merged quickly.
The unit tests were going to be the gate. This mean we were running about 800 to 1000 tests per minute. Not slow, but long enough. Any process for me as a dev
that takes long enough for me to think "Oh, I'll just do <xyz> while I wait" leads to me losing context, and usually forgetting what I was waiting on in
the first place.

## The Bottleneck
I had theories. Slow database queries, I figured. A staff engineer reached out to ask me if I saw anything obvious that would be slowing down our application
boot times. He had made the effort to convert some of our feature tests to plain old PHPUnit tests and saw major improvements in the speed. He reported that
boot times were fine in production. He's a smart guy and I trusted his intuition, so I decided to go down a rabbit hole.

Leveraging [Herd's profiler](https://herd.laravel.com/docs/macos/debugging/profiler#profiling-cli-scripts), I decided to take a look at where our time
and memory consumption was getting eaten up. What I found was a bit surprising.

In a partial run of our test suite run in series, we were spending a great deal of time and memory preparing routes and loading the configuration. My initial
suspicion about database queries wasn't proving itself to be true.

## The Design of a Modular Monolith
In order to allow for domain ownership, modules often mirror the default Laravel application structure.

```txt
- src
---- Shipping
-------- config
------------ shipping.php
-------- Http
------------ Middleware
------------ Controllers
------------ routes.php
-------- Models
-------- Provider
------------ ShippingServiceProvider.php
---- Payments
-------- config
------------ payments.php
-------- Http
------------ Controllers
---------------- ProcessPaymentController.php
------------ routes.php
-------- Models
-------- Provider
------------ PaymentsServiceProvider.php
```

You'll notice that each domain (`Shipping` and `Payments` in the above example) has its own config, routes file, and ServiceProvider. The service provider will
register the modules routes and merge in its own config.

```php
<?php

declare(strict_types=1);

namespace Domain\Shipping\Provider\ShippingProvider;

use Illuminate\Support\ServiceProvider;

final class ShippingServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->loadRoutesFrom(__DIR__ . '/../Http/routes.php');
        $this->mergeConfigFrom(__DIR__ . '/../config/payments.php');
    }
}
```

When the application begins, such as the start of a request or an artisan console command, each service provider is instantiated and its `boot()` method
is called. But Laravel's got your back. You can cache [routes](https://laravel.com/docs/12.x/deployment#optimizing-route-loading),
[config](https://laravel.com/docs/12.x/deployment#optimizing-configuration-loading), [views](https://laravel.com/docs/12.x/deployment#optimizing-view-loading), 
and [events](https://laravel.com/docs/12.x/deployment#caching-events) before deployment. That means your web requests can be served faster because the expense
of gathering routes, events, configs, and views has been paid once and is stored on disk as a PHP array.

## Back to the tests
For every "feature" test (those which extend from Laravel's `Illuminate\Foundation\Testing\TestCase`), the application is built for each test method. Take
the following trivial test as an example.

```php

use Illuminate\Foundation\Testing\TestCase;

final class MyTest extends TestCase
{
    public function test_collect_all_returns_empty_array(): void
    {
        // When
        $actual = collect()->all();

        // Then
        self::assertEquals([], $actual);
    }

    public function test_collection_can_return_count(): void
    {
        // Given
        $c = collect([1, 2, 5]);

        // Then
        self::assertEquals(3, $c->count());
    }
}
```

While these tests are pretty trivial, what I want to highlight is what happens before either one of these tests is executed. The entire application boot process
has to happen, which includes gathering routes from every service provider, as well as configuration. Not just once for the entire class, but one time for **each**
test method.
