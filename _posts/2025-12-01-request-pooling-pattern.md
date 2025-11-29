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

$flightsResponse = Http::post('https://travel-api.example.dev/flights/', [
    'from' => 'New York City, NY',
    'to' => 'San Francisco, CA',
    'departing' => '2026-02-13',
    'returning' => '2025-02-16',
]);

$hotelsResponse = Http::post('https://hotels-api.example.dev/hotels', [
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

$carsResponse = Http::post('https://vehicles-api.example.dev/rental-vehicles', [
    'location' => 'San Francisco',
    'type' => 'sport',
    'options' => [
        'orange',
        'lamborghini',
    ],
]);

$activitiesResponse = Http::get('https://around-town-api.example.dev/SF-CA-USA', [
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
| **Total** | 4.5s |

We can improve the wait time by leveraging HTTP pooling.

```php
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Http;

$responses = Http::pool(static function (Pool $pool) {
    $pool->as('flights')->post('https://travel-api.example.dev/flights/', [
        'from' => 'New York City, NY',
        'to' => 'San Francisco, CA',
        'departing' => '2026-02-13',
        'returning' => '2025-02-16',
    ]);

    $pool->as('hotels')->post('https://hotels-api.example.dev/hotels', [
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

    // ...the other requests
});
```

Now the response wait time is only as long as the slowest request.

| Request | Time |
|---------|-------|
| Flights | 1.1s |
| Hotels | 1.9s |
| Cars | 1.1s |
| Activities | 0.4s |
| **Total** | 1.9s |

**Note**: HTTP pooling doesn't solve all your problems. In many applications, a response from one endpoint
is used to inform another call in the chain. Use HTTP pooling where it fits, but recognize that the
sequential nature of some requests is a requirement.


## What Does pool() Return?

`Http::pool()` returns an array with each value being one of:

* `\Illuminate\Http\Client\ConnectionException` meaning there was a timeout trying to connect to the server
* `\Illuminate\Http\Client\Response` a response object if the request received a response
* `\Illuminate\Http\Client\RequestException` if you marked that you want your request to `throw()` on failing status codes

Let's take the last example and see how we might use the responses.

```php
use Carbon\CarbonImmutable;
use App\Schema\AvailableFlight;
use App\Schema\ServiceUnavailableResponse;

$responses = Http::pool(static function(Pool $http) { /* ... */ });

$apiResponse = [];

$flightResponse = $responses['flights'];
if ($flightResponse instanceof ConnectionException) {
    $apiResponse['flights'] = new ServiceUnavailableResponse;
} else {
    $flights = [];
    foreach($flightResponse->json()['results'] as $flightResult) {
        $flights[] = new AvailableFlight(
            airline: $flightResult['carrier'],
            airport: $flightResult['airport'],
            departure: CarbonImmutable::parse($flightResult['departing']),
            cost: (string) $flightResult['price'],
        );
    }

    $apiResponse['flights'] = $flights;
}
```

If you're like me, there's something off about putting all of this mapping logic into a single
method of a service class. I want there to be single responsibility, not because Uncle Bob told me
so, but because I want to be able to test my code at the component level. And of course, there's
that gnawing feeling that "maybe I'll need to use this in another place," but more on that later.


## Promises To the Rescue

As mentioned above, the async nature of HTTP requests is made possible thanks to Guzzle's
[Promises](https://github.com/guzzle/promises) library. Because PHP is not async by nature,
the promises offered are closer to Laravel's [Pipeline](https://laravel.com/docs/12.x/helpers#pipeline)
than they are to promises in JavaScript. They just so happen to be used by async requests.

The promise interface allows us to chain mutations together and then wait for each link in the chain
to be resolved. We can pipe our Response into a method and have it give us back a POPO (or Laravel Data
object if you fancy).

```php
namespace App\Requests;

use App\Schema\AvailableFlight;
use App\Schema\ServiceUnavailableResponse;
use Illuminate\Http\Client\ConnectionException;
use Illuminate\Http\Client\RequestException;
use Illuminate\Http\Client\Response;
use Throwable;

class GetFlights // This name will make more sense in a moment
{
    public function mapToAvailableFlight(array $flight): AvailableFlight
    {
        return new AvailableFlight(
            airline: $flight['carrier'],
            airport: $flight['airport'],
            departure: CarbonImmutable::parse($flight['departing']),
            cost: (string) $flight['price'],
        );
    }

    /**
     * @return  list<AvailableFlight>|ServiceUnavailableResponse
     */
    public function mapResponseToAvailableFlights(
        Response|RequestException|ConnectionException $flightsResponse
    ): array|ServiceUnavailableResponse {
        if ($flightsResponse instanceof Throwable) {
            return new ServiceUnavailableResponse;
        }

        $flights = [];

        foreach($flightsResponse->json()['results'] as $flight) {
            $flights[] = $this->mapToAvailableFlight($flight);
        }

        return $flights;
    }
}
```

Now we can leverage the `then()` method on our promise.

```php
$responses = Http::pool(static function (Pool $pool) {
    $pool->as('flights')
        ->post('https://travel-api.example.dev/flights/', [
            'from' => 'New York City, NY',
            'to' => 'San Francisco, CA',
            'departing' => '2026-02-13',
            'returning' => '2025-02-16',
        ])
        ->then(
            (new GetFlights)->mapResponseToAvailableFlights(...)
        );

    /* ... */
});

if ($responses['flights'] instanceof ServiceUnavailableResponse) {
    // ... handle the failure
} else {
  // now we have an array of AvailableFlight
}
```

## Making It More Extensible Still

I conspicuously named the class above `GetFlights` because I want to highlight my favorite part of
this pattern. It's quite common at my work that I need to need to make a one-off request to fetch
some data, but at other times, it's more practical to do so in a pool. This can lead to code
duplication, which can lead to drift: I updated a parameter in this service method, but
forgot to do in a different service method where maybe I am pooling the requests.

The HTTP facade allows us to mark a single request as async, even if it's not being used in a pool.
Then our terminal function (like `post()` or `get()`) returns a PromiseInterface, rather than
a Response object.

```php
$bodySize = Http::async()
    ->get('https://cosmastech.com')
    ->then(fn (Response $response) => strlen($response->getBody()))
    ->wait();
```

In the above, we are making a single request that will return the character count of a webpage.

So how can we use this as a lever for better devex and eliminating duplication? Let's add a few more
methods to our GetFlights class.

```php
use GuzzleHttp\Promise\PromiseInterface;
use Illuminate\Http\Client\PendingRequest;
use Illuminate\Support\Facades\Http;

class GetFlights
{
    public function fromPendingRequest(
        array $flightRequestBody,
        ?PendingRequest $pendingRequest = null
    ): PromiseInterface {
        $pendingRequest ??= Http::createPendingRequest();

        return $pendingRequest
            ->async()
            ->post('https://travel-api.example.dev/flights/', $flightRequestBody)
            ->then($this->mapResponseToAvailableFlights(...));
    }

    public function fetch(array $flightRequestBody): array
    {

    }
```
