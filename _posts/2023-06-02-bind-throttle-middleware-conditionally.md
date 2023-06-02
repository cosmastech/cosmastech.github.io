---
title: "Bind ThrottleRequestsWithRedis conditionally"
date: 2023-06-02
tags: Laravel, redis
---

The [Laravel docs](https://laravel.com/docs/10.x/routing#throttling-with-redis) note that if you are using Redis for your caching layer, ThrottleRequestsWithRedis is a better option for the throttling middleware.

But if you don't want to make Redis a requirement for installing and running your application, an easy fix is to just conditionally bind the special Redis version in the container.

```php
use Illuminate\Routing\Middleware\ThrottleRequests;
use Illuminate\Routing\Middleware\ThrottleRequestsWithRedis;

    public function boot(): void
    {
        // if using Redis for cache, use the Redis optimized ThrottleRequests middleware
        $rateLimiterStore = $this->app->make('cache')
            ->driver($this->app['config']->get('cache.limiter'))
            ->getStore();

        if ($rateLimiterStore instanceof RedisStore) {
            $this->app->bind(ThrottleRequests::class, ThrottleRequestsWithRedis::class);
        }
    }
```

Now if the environment supports a Redis rate limiter, ThrottleRequestsWithRedis will be used in place of ThrottleRequests. Otherwise, the base ThrottleRequests class is used. 😃
