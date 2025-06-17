---
title: "FailOnException: Short-circuit Laravel job retries"
date: 2025-06-17
tags: [Laravel, PHP, queues]
description: A simple way to stop a job when further attempts will fail.
---

## The Problem
We've all had this problem. We have a queued job that grabs some data from the database, performs some checks on it, and then inserts a new record or calls out to an external API.
We want the job to retry if the API request fails or there's a hiccup with writing to the database, however, if any of the checks fail, we want to mark the job as failed and not
bother retrying it.

Not seeing anything obvious in the Laravel docs, we reach for Claude to tell us how to do this. It hallucinates some methods that don't exist, and now we have our original problem
and we're a little pissed off with AI.

**We've all been there.**

But with the release of [Laravel 12.19](https://github.com/laravel/framework/releases/tag/v12.19.0), there's an easier way.

## `FailOnException` Job Middleware
The solution is now as easy as simply adding a middleware that specifies which exceptions short-circuit the job retry.

For instance, consider your application receives an SNS payload and has to fetch a user from another system and store a record in your local database. You might have a job that looks like this:

```php
use App\Actions\RetrieveUserFromApiAction;
use App\Exceptions\UserDoesNotExistException;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Http\Client\ConnectionException;
use Illuminate\Http\Client\RequestException;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class SyncUserJob implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    public $tries = 10;

    public function __construct(private readonly array $snsPayload)
    {
    }

    /**
     * @throws UserDoesNotExistException
     * @throws ConnectionException
     * @throws RequestException
     */
    public function handle(RetrieveUserFromApiAction $retrieveUserAction): void
    {
        $user = $retrieveUserAction->handle($this->snsPayload['user_id']);

        // do some other stuff with the user based on the SNS event...
    }
}
```

You'll notice that our docblocks indicate that the job's `handle()` method can throw a host of exceptions. A `RequestException` or `ConnectionException` can occur due to some kind of transient network issue,
or perhaps the external API is having an outage. However, if the `UserDoesNotExistException` is thrown, then further retries will return the exception again and again.

Instead of putting unnecessary strain on our infrastructure, we can inform the queue worker to mark this job as failed by leveraging the `Illuminate\Queue\Middleware\FailOnException` middleware. We simply
add a middleware method to our job:

```php
    public function middleware(): array
    {
        return [
            new FailOnException([UserDoesNotExistException::class]),
        ];
    }
```

When the exception is thrown, the job will be immediately marked as failed, no longer ticking down tries. This exception will be visible in Horizon (or Telescope, or your APM) as the failure reason.
