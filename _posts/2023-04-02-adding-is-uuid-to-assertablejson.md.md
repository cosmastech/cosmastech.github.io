---
title: "Add isUuid() to AssertableJson"
date: 2019-01-20
tags: Laravel, Testing
---

AssertableJson is a huge time saver when it comes to feature tests within Laravel. However, the AssertableJson@whereType functionality only allows for built-in PHP types.

What if we want to assert that something is a valid UUID?  We can simply add a Macro to the AssertableJson object.

```php
use Illuminate\Support\ServiceProvider;
use Illuminate\Testing\Fluent\AssertableJson;
use Str;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        if ($this->app->runningUnitTests()) {
            AssertableJson::macro(
                'isUuid',
                fn (string $key) => $this->where($key, fn ($value) => Str::isUuid($value))
            );
        }
    }
}
```

Now within a test, you can use `isUuid()`

```php
class UuidResponseTest extends TestCase
{
    public function testUuidInResponse()
    {
        $this->getJson('/api/path-to-an-endpoint')
          ->assertJson(fn(AssertableJson $json) => $json->isUuid('data.user.uuid'));
    }
}
```
