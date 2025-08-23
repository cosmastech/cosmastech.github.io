---
title: "Creating type-safe configs in Laravel"
date: 2025-08-23
tags: [Laravel, config, PHP]
description: A pattern that I like to use for type-safe configs.
---

When I began building a new product in Laravel last year, I wanted a lot of things: good test coverage, type-safety, a high PHPStan level, and for some reason, to write the project like I was still working on Java Spring Boot microservices. I’ll leave that last one for another time, as it was a complete disaster, but I did learn a lot from it.

One of the things that feels yucky about Laravel is the use of magic strings. When PHP added enums, they immediately changed the way I write code. Recently, Laravel has accepted a number of PRs that allow passing enums to methods which previously only accepted primitives. For instance, to retrieve a value from a file defined in the `config/` directory, you make a call `config('github.api_token')` which returns _something_ of some type, and if it's not found, it returns null. For additional type-safety, you can leverage `Config::string('github.api_token')`, functionality that was added in Laravel 11.x.

One of the patterns that I landed on which feels elegant is creating a class for a service's configuration. Here's a little configuration for an AIM client (a little nostalgia for those who remember crafting the perfect away message).

```php
namespace App\Config;

use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\Singleton;

#[Singleton]
final readonly class AolInstantMessengerConfig
{
    /**
     * @param array<int, string> $scopes
     */
    public function __construct(
        #[Config('aim.api_token')]
        public string $apiToken,
        #[Config('aim.base_url')]
        public string $baseUrl,
        #[Config('aim.scopes')]
        public array $scopes,
        #[Config('aim.timeout', 60)]
        public int $timeout,
    ) {
    }
}
```

Let's take a look at what's going on in this example, and how we might use this configuration.

### How we can use this
In our code, we will never call `config('aim.api_token')`, but instead we always refer to these values via the config class we have created. This gives us type-safety and reduces magic strings in our code.

Historically, I would write something like:

```php
use Http;

Http::baseUrl(config('aim.base_url'))
    ->timeout(config('aim.timeout'))
    ->withToken(config('aim.api_token'))
    ->post('/login', [
        'scopes' => config('aim.scopes'),
    ]);
```

Now instead, I can just reference a configuration object.
```php
use App\Config\AolInstantMessengerConfig;
use Http;

$config = app(AolInstantMessengerConfig::class);

Http::baseUrl($config->baseUrl)
    ->timeout($config->timeout)
    ->withToken($config->apiToken)
    ->post('/login', [
        'scopes' => $config->scopes,
    ]);
```

### The `Config` attribute
The `Config` attribute tells the Laravel container to inject the value from the `config/aim.php` file. Paired with constructor property promotion, this makes the class construction nice and clean.

### The `Singleton` attribute
This attribute, added by [Rias](https://rias.be/) and made available in the Laravel 12.21 release, will bind a class as a singleton in the container when it is first resolved. When there is a request to build this class subsequently, it will just reference the same instance of the `AolInstantMessengerConfig`. 

For the toy example above, I wouldn't expect a noticeable change in performance if it was bound as a singleton or not. That said, this attribute is a nice and convenient way to register a singleton.

### `readonly` class
Marking this class as readonly ensures that these properties never get changed after instantiation. While I imagine (hope?) that most of you are not modifying config values in your production code, this signals to users that "this is what it is and you shall not change it."

This pairs nicely with making the class a singleton: we know that the configuration at first instantiation will never change in the course of the app.


## But what about ...
There are tons of ways to skin this cat. You may prefer calling `Config::string()` or `Config::boolean()` for type-safety. You may not like having a separate class for configuration. You may not mind using magic strings. That's fine! Nothing here is objectively better or worse: it's entirely team and developer dependent. Just wanted to share something that I enjoy.

Have fun building!