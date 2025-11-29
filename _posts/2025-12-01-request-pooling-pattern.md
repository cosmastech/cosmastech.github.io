---
title: "Leveraging Promises and HTTP Pooling"
date: 2025-12-01
tags: Laravel, async, pooling, HTTP
---

Laravel 8 first introduced [HTTP request pooling](https://laravel.com/docs/12.x/http-client#request-pooling), thanks
to a contribution from [Andrea Marco Sartori](https://github.com/cerbero90). This allows developers to write code
which will execute any number of HTTP requests concurrently. Under the hood, this is made possible thanks to the
[async request functionality of Guzzle](https://docs.guzzlephp.org/en/stable/quickstart.html#async-requests) and
[cURL's multi handler functionality](https://www.php.net/manual/en/function.curl-multi-init.php).

Let's imagine we are building a platform for travels to get the best deals on travel. A traveler needs
to transportation, lodging, a rental vehicle, and recommendations for what to do when they are in town.

```php
use Illuminate\Support\Facades\Http;

$flights = Http::post('https://travel-api.example.dev/flights/', [
    'from' => 'New York City, NY',
    'to' => 'San Francisco, CA',
    'departing' => '2026-02-13',
    'returning' => '2025-02-16',
]);

$hotels = Http::post('https://hotels-api.example.dev/hotels', [
    'location' => 'San Francisco',
    'check_in' => '2026-02-13',
    'check_out' => '2026-02-16',
    'adults' => 2,
    'amenities' => [
        'pool' => true,
        'breakfast' => false,
        'shuttle' => false,
    ],
]);

$cars = Http::post('https://vehicles-api.example.dev/rental-vehicles', [
    'location' => 'San Francisco',
    'type' => 'sport',
    'options' => [
        'orange',
        'lamborghini',
    ],
]);

$activities = Http::get('https://around-town-api.example.dev/SF-CA-USA', [
    'categories' => [
        'nightlife',
        'art',
        'music',
        'historical',
    ],
]);
```

This above example gathers all of this data, but it does so sequentially. If this data is all gathered
during a web request, the caller has to wait for the cumulative time of each request.

| Request | Time |
|---------|-------|
| Flights | 1.1s |
| Hotels | 1.9s |
| Cars | 1.1s |
| Activities | 0.4s |
|=|=
| Total | 4.5s |
