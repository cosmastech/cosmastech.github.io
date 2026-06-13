---
title: "TIL: Identifying Exceptions in Laravel Middleware"
date: 2026-06-13
tags: Laravel, middleware, http, testing
---

My team and I have been working on building a light observability layer around an important API in our Laravel app. We had crafted the functionality: data models to represent what happens during an event, new infra to keep our core database free from noisy writes, and an AI-assisted planning doc that was too long to read. We were doing software engineering in 2026. :sunglasses:

One of the key fields to capture in the DB was exceptions thrown from a controller. In Laravel, fatal exceptions get caught and rendered to the client as JSON through the magic of the framework. We wanted to be able to record this exception distinctly in our database.

In order to push observability to the edge so that we didn't need to instrument each existing controller or service, we were going to capture everything through a middleware. In my mind, it would look something like this:

```php
class CaptureApiRequest
{
    public function __construct(private Recorder $recorder)
    {
        //
    }

    public function handle(Request $request, Closure $next)
    {
        $this->recorder->captureRequest($request);

        try {
            return $next($request);
        } catch (Throwable $e) {
            $this->recorder->captureException($e);

            throw $e;
        }
    }
}
```

Simple, right? If an exception was thrown inside of the controller, we would capture it, re-throw it, and let the container handle it. ...right?

## Well, Akshually

This didn't work. We added a dummy exception in a controller method, hit the endpoint, got back the error we expected but nothing was recorded.

It turns out that my mental model about how exceptions work in the Laravel request lifecycle was wrong. The [routing Pipeline](https://github.com/laravel/framework/blob/13.x/src/Illuminate/Routing/Pipeline.php#L40-L58) is built when a request is received. The controller is identified, as well as all of the middleware for the route, and then the pipeline passes the request through the middleware, into the controller, and resolves a response.

It is the routing Pipeline, not the container that handles converting a thrown exception into a response (JSON or HTML, depending on the accepted content type).

If an exception is thrown by middleware or within a controller, the Pipeline will create or modify the response. What is "the response" though? Well, it can be anything, literally. But most often, it is a Response, JsonResponse, or RedirectResponse. All of which apply the `ResponseTrait`. And this is actually where the thrown exception lives, and how you can access it.

## `ResponseTrait`

This [trait](https://github.com/laravel/framework/blob/13.x/src/Illuminate/Http/ResponseTrait.php) has a handful of methods and properties on it. Here's an excerpt of the code to focus on:

```php
trait ResponseTrait
{
    // ...omitted code...

    /**
     * The exception that triggered the error response (if applicable).
     *
     * @var \Throwable|null
     */
    public $exception;

    // ...omitted code...

    /**
     * Set the exception to attach to the response.
     *
     * @param  \Throwable  $e
     * @return $this
     */
    public function withException(Throwable $e)
    {
        $this->exception = $e;

        return $this;
    }
}
```

This trait has a simple setter method `withException()` and the `$exception` itself is public.

Inside of the routing `Pipeline`, there is a method called `handleException()`. When any step of the Pipeline throws an Exception, this code gets called:

```php
protected function handleException($passable, Throwable $e)
{
    if (! $this->container->bound(ExceptionHandler::class) ||
        ! $passable instanceof Request) {
        throw $e;
    }

    $handler = $this->container->make(ExceptionHandler::class);

    $handler->report($e);

    $response = $handler->render($passable, $e);

    if (is_object($response) && method_exists($response, 'withException')) {
        $response->withException($e);
    }

    return $this->handleCarry($response);
}
```

When an exception is caught, the framework's `ExceptionHandler` will report the exception and then convert that into a response. If `$response` happens to have a `withException()` method, the framework stores the exception on the newly created response. The Pipeline doesn't exit early because of an exception.

So, if our middleware is trying to capture exceptions (`ValidationException`s or whatever is thrown inside of the controller itself), a try-catch block doesn't cut it. We need to instead grab the `$exception` property from the response.

## A Solution

So instead of a try-catch block, let's see if the response has an exception associated with it.

```php
public function handle(Request $request, Closure $next)
{
    $this->recorder->captureRequest($request);

    $response = $next($request);

    /*
     * If the response is an object with an exception property that
     * happens to be set and an instance of Throwable, let's record
     * that in our observability database.
     */
    if (
        is_object($response)
        && property_exists($response, 'exception')
        && ($exception = $response->exception ?? null) instanceof Throwable
    ) {
        $this->recorder->captureException($exception);
    }

    return $response;
}
```

With that, we started seeing the exceptions logged in our database! :celebrate:

### For the Pedants

It might still make sense to have a try-catch block, in case there's some kind of failure in the middleware Pipeline. But for simplicity's sake, I'm only including the part that's important for 98% of use-cases.

## Takeaways

I wanted to capture this experience because it was something as Laravel developer I had never come across.

First, I learned that Laravel includes the exception within a response separate from the rendering. This feels like it could unlock some interesting test scenarios where instead of asserting just against the JSON response, I could also assert against the thrown exception itself.

Secondly, manual testing is as important as ever. For our original implementation, we may have had a passing test like this.

```php
public function test_it_records_exceptions(): void
{
    $next = fn (): never => throw new Exception('Boom');
    $middleware = $this->app->make(CaptureApiRequest::class);

    try {
        $middleware->handle($this->app->make('request'), $next);
        $this->fail('No exception was thrown');
    } catch (Exception $e) {
        $this->assertSame('Boom', $e->getMessage());
    }

    $this->assertCount(1, DB::table('exceptions')->get());
}
```

This would provide false confidence, as in practice, that's not how the middleware would be executed. Manual testing proved our expectations faulty, and instead, an integration test was written to insulate us against regressions.
