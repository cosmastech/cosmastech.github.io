---
title: "How WithoutRelations can keep your code clean"
date: 2023-08-15
tags: Laravel, queue, 
description: How to make use of the new WithoutRelations attribute on jobs and events.
---

If you want your Laravel application to be as efficient as possible, you're probably already calling `withoutRelations()` within [the constructor of your Jobs and Events](https://laravel.com/docs/10.x/queues#handling-relationships).

With the release of Laravel 10.19, there's a new attribute called [WithoutRelations](https://github.com/laravel/framework/blob/b8557e4a708a1bd2bc8229bd53feecfa2ac1c6fb/src/Illuminate/Queue/Attributes/WithoutRelations.php) you can apply on individual properties or even a whole job class.

Let's first take a look at how you might be doing this today.

```php
use App\Models\User;
use App\Models\Subscription;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ExtendSubscription implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    private readonly User $user;

    private readonly Subscription $currentSubscription;

    public function __construct(
        User $user,
        Subscription $currentSubscription,
        private readonly int $numberOfDays = 30
    ) {
        $this->user = $user->withoutRelations();
        $this->currentSubscription = $currentSubscription->withoutRelations();
    }

    public function handle()
    {
        // ...
    }
}
```

This is fine, but I know that I love a clean constructor: relies on property promotion, marking properties as readonly if I know that I won't be changing them.

Now let's see how we can leverage WithoutRelations here.

```php
use Illuminate\Queue\Attributes\WithoutRelations;

    public function __construct(
        #[WithoutRelations]
        private readonly User $user,
        #[WithoutRelations]
        public readonly Subscription $currentSubscription,
        private readonly int $numberOfDays = 30
    ) {
    }
```

😍 Beautiful!

We can take this a step further and apply `WithoutRelations` to the whole Job.

```php
#[WithoutRelations]
class ExtendSubscription implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        private readonly User $user,
        public readonly Subscription $currentSubscription,
        private readonly int $numberOfDays = 30
    ) {
    }
```

🧹 Now that's clean.

### In Closing
The `WithoutRelations` attribute is very handy and can make your codebase more clean.

You can leverage the attribute on any property of a class that that employs `SerializesModels` like Events or Jobs.
