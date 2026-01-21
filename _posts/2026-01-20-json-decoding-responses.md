---
title: "JSON Decoding HTTP Responses"
date: 2026-01-20
tags: Laravel
---

With the release of [Laravel 12.48.0](https://github.com/laravel/framework/releases/tag/v12.48.0), you can now specify which flags you want to use when decoding an HTTP response from the HTTP client.

```php
$response = Http::get('https://cosmastech.com'); // a URL that returns HTML
$json = $response->json();
```

What is the value of `$json` here? You may be surprised to find out that it's `null`. Why? Because by default, that's how PHP treats invalid JSON passed to `json_decode()`. If you're just now learning this esoteric fact, don't feel bad. Most devs probably come to learn this tidbit when their code breaks in production.

If you want to check if `json_decode()` returned `null` because the input was `'null'` or because there was an error, you can use the `json_last_error()` function.

That's pretty cumbersome, and thankfully, in PHP 8, the ability to pass `JSON_THROW_ON_ERROR` as a flag to `json_decode` was added. This will raise an exception when given a non-JSON string to parse.

```php
$str = '{"HELLO": xyz}'; // Invalid JSON
json_decode($str, flags: JSON_THROW_ON_ERROR); // throws a \JsonException
```

## How to Use In Laravel

Laravel now offers the ability to decode JSON from an HTTP response, specifying any of the [standard decoding flags](https://www.php.net/manual/en/function.json-decode.php#:~:text=Bitmask%20of%20JSON_BIGINT_AS_STRING%2C%20JSON_INVALID_UTF8_IGNORE%2C%20JSON_INVALID_UTF8_SUBSTITUTE%2C%20JSON_OBJECT_AS_ARRAY%2C%20JSON_THROW_ON_ERROR.).

```php
$response = Http::get('https://cosmastech.com');
try {
    $json = $response->json(flags: JSON_THROW_ON_ERROR);
} catch (\JsonException $e) {
    // Perfect, we KNOW we have invalid JSON
}
```

You can additionally pass these flags to a few other `Response` methods as well: `collect()`, `object()`, and `fluent()`.

```php
Http::fake([
    '*' => '{
        "hello": "world",
        "big_int": 123343343580999843483023
    }'
]);
$response = Http::get('https://laravel.com');
$obj = $response->object(
    flags: JSON_BIGINT_AS_STRING
);
var_dump($obj->big_int); // "123343343580999843483023"
```
In this example, you'll notice we passed the JSON_BIGINT_AS_STRING flag. If attempting to decode JSON with an integer that exceeds PHP_INT_MAX (9223372036854775807 on my system), it will convert the integer to a float. By using this flag, rather than converting the int to a float, it parses it as a string.

## Application Defaults

You may wish to always decode your HTTP response JSON with certain flags. This is available by adding a method like this to AppServiceProvider's boot method:

```php
use Illuminate\Http\Client\Response;

public function boot()
{
    Response::$defaultJsonDecodingFlags = JSON_BIGINT_AS_STRING | JSON_THROW_ON_ERROR;
}
```

In the above example, we don't need to pass any flags when calling `$response->json()`, it will always default to converting overflowed integers to strings and throwing an exception on invalid JSON.

If for some reason you want to override these defaults, just pass different flags at decoding time.

```php
Http::get('https://cosmastech.com')->json(flags: 0); // use default json_decode behavior
```

## Use Flags

Trying to remember to always pass these flags at every call site can be a really arduous task. If you can, try to set sensible defaults for how JSON is decoded in your Laravel application. If not, going forward, add them to new call sites. Future self will thank you for the clarity.
