---
title: "TIL: Laravel's Factory::forEachSequence"
date: 2025-01-04
tags: [Laravel, Eloquent, Testing, TIL]
description: How and why to use the forEachSequence helper
---

![Drake Knows](/assets/2025/for-each-sequence.png)

This title is a lie, I actually did not learn this today, but came across it a few weeks ago.
Nonetheless, I wanted to share it since I think it's incredibly valuable, but is not
included in the Laravel documentation.

## Factory::sequence()

When I need to create a series of Models of the same type, the tool to reach for is Laravel factories. Specifically, [the sequence](https://laravel.com/docs/11.x/eloquent-factories#sequences) functionality.

For instance, say I have Payment model. Here is an example of the factory:

```php
/**
 * @extends Factory<Payment>
 */
class PaymentFactory extends Factory
{
    protected $model = Payment::class;

    public function definition(): array
    {
        return [
            'gateway' => $this->faker->randomElement(['stripe', 'ach']),
            'amount' => $this->faker->numberBetween(1_00, 25_000_00),
            'status' => $this->faker->randomElement(['paid', 'pending', 'refunded']),
            'user_id' => User::factory(),
            'created_at' => Carbon::now(),
            'updated_at' => Carbon::now(),
        ];
    }
}
```

Say I want to create 4 payments belonging to a user. I might reach for something like this in
my test case.

```php
$user = User::factory()->create([
    'id' => 102,
    'first_name' => 'Luke',
    'last_name' => 'Kuzmish'
]);

$payments = Payment::factory()
    ->for($user)
    ->sequence(
        ['id' => 5000, 'amount' => 100_00, 'status' => 'paid'],
        ['id' => 5002, 'amount' => 200_00, 'status' => 'refunded'],
        ['id' => 7000, 'amount' => 1_000_00, 'status' => 'pending'],
        ['id' => 7002, 'amount' => 1_050_00, 'status' => 'pending'],
    )
    ->create(['gateway' => 'stripe']);
```

What is the value of `$payments`? You may be frustrated to learn that it's actually just a
single `Payment` model, not a Collection of four Payments.

So what did I forget? I forgot to specify the count of models to create.

```diff
$payments = Payment::factory()
    ->for($user)
    ->sequence(
        ['id' => 5000, 'amount' => 100_00, 'status' => 'paid'],
        ['id' => 5002, 'amount' => 200_00, 'status' => 'refunded'],
        ['id' => 7000, 'amount' => 1_000_00, 'status' => 'pending'],
        ['id' => 7002, 'amount' => 1_050_00, 'status' => 'pending'],
    )
++  ->count(4)
    ->create(['gateway' => 'stripe']);
```

I've personally made this mistake countless times. Worse still is I may remember to chain
`count()` but later modify the test to add a new sequence entry.

```diff
$payments = Payment::factory()
    ->for($user)
    ->sequence(
        ['id' => 5000, 'amount' => 100_00, 'status' => 'paid'],
        ['id' => 5002, 'amount' => 200_00, 'status' => 'refunded'],
        ['id' => 7000, 'amount' => 1_000_00, 'status' => 'pending'],
        ['id' => 7002, 'amount' => 1_050_00, 'status' => 'pending'],
++      ['id' => 9999, 'amount' => 2_999_99, 'status' => 'refunded'],
    )
    ->count(4)
    ->create(['gateway' => 'stripe']);
```

Here I end up with only four Payment models again.

## A Better Solution

Enter `forEachSequence`. Using this method, the factory will create a Payment model for
each sequence entry.

```php
$payments = Payment::factory()
    ->for($user)
    ->forEachSequence(
        ['id' => 5000, 'amount' => 100_00, 'status' => 'paid'],
        ['id' => 5002, 'amount' => 200_00, 'status' => 'refunded'],
        ['id' => 7000, 'amount' => 1_000_00, 'status' => 'pending'],
        ['id' => 7002, 'amount' => 1_050_00, 'status' => 'pending'],
        ['id' => 9999, 'amount' => 2_999_99, 'status' => 'refunded'],
    )
    ->create(['gateway' => 'stripe']);
```

Now we have our five Payment models and never need to worry about specifying the count of 
models to create.


User::factory()
    ->forEachSequence(
        ['name' => 'Taylor Otwell'],
        ['name' => 'Nuno Maduro'],
        ['name' => 'Christoph Rumpel'],
    )
    ->create();