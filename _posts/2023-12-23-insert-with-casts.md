---
title: "Builder@insertWithCasts() -  Bulk inserts with model casting"
date: 2023-12-23
tags: Laravel, Eloquent
description: A handy function for inserting many records using Eloquent while still performance attribute casting
---

## A problem

If you want to insert multiple records using Eloquent, you have a few options:

###  `Model::insert()`
This performs a single INSERT query, however, you have to provide the exact parameters you want persisted to your database.
This means things you must manually set timestamps, as well as handle the juggling your model's casts may have.

```php
User::insert([
  ['name' => 'Jim', 'created_at' => $now = now(), 'updated_at' => $now],
  ['name' => 'Ed', 'created_at' => $now, 'updated_at' => $now],
  // ...
]);
```

This can get messy if you're using casts on your model, as you'll need to ensure that you're converting all of the elements into the exact same format your Eloquent model would normally handle for you.

```php
use Illuminate\Database\Eloquent\Model;

enum Role: string
{
    case ADMIN = 'admin';
    case WRITER = 'writer';
    case GUEST = 'reader';
}

class User extends Model
{
    public $casts = [
        'role' => Role::class,
        'settings' => 'json',
    ];
}
```

Now you might need something like this:

```php
User::insert([
  ['name' => 'Jim', 'role' => Role::ADMIN->value, 'settings' => json_encode(['background_color' => '#fff']), 'created_at' => $now = now(), 'updated_at' => $now],
  ['name' => 'Ed', 'role' => Role::GUEST->value, 'settings' => json_encode([]), 'created_at' => $now, 'updated_at' => $now],
  // ...
]);
```

### `Model::create()`
This only creates 1 model at a time. If you have an array of records you want to create, you need to create loop through the array and perform a separate insert on each.

```php
$usersToCreate = [
  ['name' => 'Jim', 'role' => Role::ADMIN, 'settings' => ['background_color' => '#fff']],
  ['name' => 'Bob', 'role' => Role::GUEST, 'settings' => []],
  // ...
];

foreach($usersToCreate as $user) {
    User::insert($user);
}
```

For each record, you're performing a separate INSERT query. If you have hundreds of records to insert, or a very large database table, this can be time-expensive.

## A solution
Wouldn't it be great to be able to insert an array, have all of casting managed for you, and have it done in a single query?

First, let's add a custom builder to our model

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

use App\Builders\CustomEloquentBuilder;

class User extends Model
{
    // ...

    public function newEloquentBuilder($query): CustomEloquentBuilder
    {
        return new CustomEloquentBuilder($query);
    }
}
```

Now, let's make our `CustomEloquentBuilder` class.

```php
namespace App\Builders;

use Illuminate\Database\Eloquent\Builder;

class CustomEloquentBuilder extends Builder
{
    public function insertWithCasts(iterable $values): bool
    {
        if (empty($values)) {
            return true;
        }

        if (! is_array(reset($values))) {
            $values = [$values];
        }

        $modelInstance = $this->newModelInstance();
        $timestampColumns = [];

        if ($modelInstance->usesTimestamps()) {
            $now = $modelInstance->freshTimestamp();

            if ($createdAtColumn = $modelInstance->getCreatedAtColumn()) {
                $timestampColumns[$createdAtColumn] = $now;
            }
            if ($updatedAtColumn = $modelInstance->getUpdatedAtColumn()) {
                $timestampColumns[$updatedAtColumn] = $now;
            }
        }

      $this->model->unguarded(function () use (&$values, $timestampColumns) {
          foreach ($values as $key => $value) {
              $values[$key] = $this->newModelInstance(array_merge($timestampColumns, $value))->getAttributes();
          }
      });

      return $this->toBase()->insert($values);
    }
}
```

### Let's put it to use

Now we can get the best of both worlds.

```php
$usersToCreate = [
    ['name' => 'Jim', 'role' => Role::ADMIN, 'settings' => ['background_color' => '#fff']],
    ['name' => 'Bob', 'role' => Role::GUEST, 'settings' => []],
    // ...
];
\DB::enableQueryLog();
User::insertWithCasts($usersToCreate);
$outputQueries = \DB::getRawQueryLog();

count($outputQueries) === 1; // true

$user = User::latest()->first();

echo $user->role === Role::GUEST; // true
echo $user->created_at instanceof \DateTime; // true
```

If you prefer, you can [find the the Eloquent builder here as a gist](https://gist.github.com/cosmastech/bfd6d060df602d3fed1f3982febb5305).
