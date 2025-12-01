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


## Serially Executed Requests Vs Pooled Requests

Let's imagine we are building a platform for travelers to get the best deals on travel. A traveler needs
transportation, lodging, a rental vehicle, and recommendations for what to do when they are in town.

```php
use Illuminate\Support\Facades\Http;

$flightsResponse = Http::post('https://travel-api.example.dev/flights/', [
    'from' => 'New York City, NY',
    'to' => 'San Francisco, CA',
    'departing' => '2026-02-13',
    'returning' => '2026-02-16',
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

The above example gathers all of this data, but it does so sequentially. If this data is all gathered
during a web request, the caller has to wait for the cumulative time of all requests. For instance,
imagine this is the response time for each:

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
        'returning' => '2026-02-16',
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
}, concurrency: 4);
```

Now the response wait time is only as long as the slowest request because

| Request | Time |
|---------|-------|
| Flights | 1.1s |
| Hotels | 1.9s |
| Cars | 1.1s |
| Activities | 0.4s |
| **Total** | 1.9s |


### Notes

The second parameter to `Http::pool()` is named `concurrency` and it informs how many requests should
be in flight at any given time. If you have 10 requests with `$concurrency` set to 5, the sixth request
will not start until the first is complete.


#### Silver Bullet?

HTTP pooling doesn't solve all your problems. In many applications, a response from one endpoint
is used to inform another call in the chain. Use HTTP pooling where it fits, but recognize that
some requests must still be executed in series.


## What Does pool() Return?

`Http::pool()` returns an array with each value being one of:

* `Illuminate\Http\Client\ConnectionException` meaning there was a timeout trying to connect to the server
* `Illuminate\Http\Client\Response` a response object if the request received a response
* `Illuminate\Http\Client\RequestException` if you marked that you want your request to `throw()` on failing status codes

Let's take the last example and see how we might use the responses.

```php
namespace App\Schema;

use Carbon\CarbonImmutable;

final readonly class AvailableFlight
{
    public function __construct(
        public string $airline,
        public string $airport,
        public CarbonImmutable $departure,
        public string $cost,
    ){
        // ...
    }
}
```

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

What if I had a class that was responsible for mapping Laravel's response into my data object?
I could then test this function in isolation, giving me confidence it behaves as desired given
different scenarios, such as "what if a key is missing? what if the connection times out? what
if the API returns a 500 response code?"


## Promises To the Rescue

As mentioned above, the async nature of HTTP requests is made possible thanks to Guzzle's
[Promises](https://github.com/guzzle/promises) library. Because the PHP runtime is not async by nature,
the Promises offered are closer to Laravel's [Pipeline](https://laravel.com/docs/12.x/helpers#pipeline)
than they are to Promises in JavaScript. While Guzzle can execute HTTP requests concurrently, Promises
can be used to simply pipe the results of one function into another.

The Promise interface allows us to chain mutations together and then wait for each link in the chain
to be resolved. For instance, we can pipe our Response into a method and have it give us back a POPO (or
[Laravel Data](https://spatie.be/docs/laravel-data/v4/introduction) object if you fancy).

```php
namespace App\Requests;

use App\Schema\AvailableFlight;
use App\Schema\ServiceUnavailableResponse;
use Carbon\CarbonImmutable;
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

Now we can leverage the `then()` method on our Promise. The `then()` method executes a callback against
the Response.

```php
$responses = Http::pool(static function (Pool $pool) {
    $pool->as('flights')
        ->post('https://travel-api.example.dev/flights/', [
            'from' => 'New York City, NY',
            'to' => 'San Francisco, CA',
            'departing' => '2026-02-13',
            'returning' => '2026-02-16',
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
    ->then(fn (Response $response) => strlen($response->body()))
    ->wait();
```

In the above, we are making a single request that will return the character count of a webpage.

Above I mentioned how we may want to use our request building and mapping logic in another place. So how can
we use this as a lever for better devex and eliminating duplication? Let's add a few more methods to our
GetFlights class.

```php
use GuzzleHttp\Promise\PromiseInterface;
use Illuminate\Http\Client\PendingRequest;
use Illuminate\Support\Facades\Http;
use RuntimeException;

class GetFlights
{
    /**
     * @param  array<string, mixed>  $flightRequestBody
     */
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

    /**
     * Retrieve available flights.
     *
     * @param  array<string, mixed>  $flightRequestBody
     * @return list<AvailableFlight>
     *
     * @throw RuntimeException when there is request failure
     */
    public function fetch(array $flightRequestBody): array
    {
        $result = $this->fromPendingRequest(
            $flightRequestBody
        )->wait();

        if ($result instanceof ServiceUnavailableResponse) {
            throw new RuntimeException('Service unavailable');
        }

        return $result;
    }

    /* ... */
}
```

With these simple additions, we can get our flight via pooling or as a one-off, and it
will always be immediately mapped to our AvailableFlight data object.

```php
$flightGetter = new GetFlights;

$requestPayload = [
    'from' => 'Erie, PA',
    'to' => 'Little Rock, AR',
    'departing' => '2026-01-11',
    'returning' => '2026-02-13',
];

// Get in a pool
$responses = Http::pool(function (Pool $pool) use ($flightGetter, $requestPayload) {
    $flightGetter->fromPendingRequest(
        $requestPayload,
        $pool->as('flights')
    );

    /* ... other pooled requests ... */
}, concurrency: 4);

// Or use it as a one-off
$availableFlights = $flightGetter->fetch($requestPayload);
```


### Why Does This Work?
HTTP Pooling works by keeping an array of PendingRequest objects, all of which are have their async
property set to true by default. When the pool() method executes, it is just awaiting an array of Promises,
for which we already chained a `then()` method to map them into the object we want. For our one-off
request case (`GetFlights@fetch()`), we are creating a new PendingRequest and marking it as async
via `$pendingRequest->async()`. We do this not because the request will be handled concurrently with
other requests, but because we want to share the Promise chaining.


## Why?

I came to this pattern as I was refactoring some endpoints which were very slow. The sluggishness was
a result of sequential requests to an external API. When these methods were written initially, they
worked fine, because we didn't need to gather everything at once. The frontend of the web application
called a separate application endpoint for flights, for hotels, for cars, etc. In that way, they were
able to be called asynchronously.

As we move towards a single endpoint returning all of gathered data, the series of requests
becomes a bottleneck. But for a product which releases code multiple times per day, and for
whom some service methods still needed to be used one-by-one, this feels like an elegant solution:
we have one class which is responsible for building a request and mapping it to a data transfer object,
but the request can be made in a batch or one-by-one.

The approach to refactoring is first to create each `Get*` class. Next we will move our existing
service methods to call this class using the `fetch()` method. Finally, we seek out opportunities
for pooling, and in those cases, refactor to the `fromPendingRequest()` method inside of an HTTP pool
rather than calling the service method.


# In Closing

When I was working towards this pattern, I felt like I had just discovered some kind of magic. PHP
can be called anachronistic for its runtime model of one process per request. However, it still offers
excellent developer ergonomics: we don't have to think about threads, function coloring, or manually
cleaning up the application at the end of a request. The ability to pool our HTTP requests to avoid
sequential slowness is a big win for developers.

---
Are you using HTTP pooling in your application? Got big thoughts on Promises? Did I make a mistake
in this post and you want to bring it to my attention? Drop a comment below or find me on [X](https://x.com/cosmastech).
