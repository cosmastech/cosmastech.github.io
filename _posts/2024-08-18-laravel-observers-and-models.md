---
title: "The Pitfalls of Events and Laravel Observers in Large Teams"
date: 2024-08-18
tags: Laravel, events, Eloquent, code architecture
description: My experience with the pitfalls of Observers
---
I want to talk about one of the biggest pain points I’ve experienced as a developer working on a large Laravel project: Model observers and events. They have introduced significant complexity and challenges on projects, multiplied by the fact that at least ten people have worked on the codebase over the span of several years.

## Observers

I initially embraced the idea of "event-driven design" a few years back. I introduced Observers into a codebase which had grown from a single developer to a team of five over the course of 3 years. The reasons for doing this were twofold.

First, I wanted to play with Observers.  I was excited to be learning so many things from reading the docs, step debugging through packages the project used, and blog posts.  It was a bit of "shiny thing syndrome."  Secondly, I thought "I want to make it so that `{some action}` always occurs when `{this model gets saved|updated|deleted}` without having to find all of the places in the code where we call `save()` or `update()`."

There are a few problems with this.

### Obscurity, Discoverability, and Cognitive Load

When I'm working on a project by myself, it's easy for me to remember that there's something happening every time I modify a model. When the team grows, features getting added or changed, the cognitive load on developers increases as they must consider the hidden side effects of model updates. This has led to things in our code like:

```php
/**
 * Stripe profile will be created or updated for a user via CreateStripeProfileJob and we'll also
 * update all any existing invoices to match their new details.
 * 
 * @see UserObserver::saved()
 * @see UserObserver::created() 
 */
$user->save();
```

### Changing Tests

Everything is handled synchronously in our test suite. If we have 100 tests that save a User in some capacity, they all need to accommodate this newly added observer. Say we have test suite that includes a test like this

```php
use App\Http\Controllers\UserController;
use App\Models\User;
use Tests\TestCase;

class UserControllerTest extends TestCase
{
    // ... other tests ...

    public function test_canUpdateUserProfile(): void
    {
        $user = User::factory()->create(['business_name' => 'foo']);

        $this->actingAs($user)
            ->postJson(
                action([UserController::class, 'update']),
                ['business_name' => 'bar']
            );

        $user->refresh();

        $this->assertEquals('bar', $user->business_name);
    }
}
```

And now we're adding an observer that updates a Stripe account if one exists or creates one if not, as well as update any invoices to they've created to match the new details. This test, which previously was testing simple CRUD functionality of a controller, will now execute the observer code as well. This makes the test take longer and could potentially fail because it was not set up to use a dummy StripeService.

We have a few options. First, we can call `createQuietly()` instead of `create()` on our UserFactory.  This is a workable solution but it's not perfect. It's possible that the `UserObserver@created()` method has some other functionality you still need to execute.

Another option would be to call `Queue::fake([CreateStripeProfileJob::class])`. But what if you later decide you need to dispatch another job inside of your Observer? You'll have to also fake that job, and do so in every test that creates or updates a user.

The underlying issue is that changing unrelated tests signals a deeper problem in how well our tests isolate and validate specific functionality. How good are my tests if they are not isolated from out of process functionality?  Can I be confident in them?  Or do I need to perform a lot of manual testing?  What if I add something to the observer that doesn't cause the test to break?  Have I really written a good test?

### Not Triggered By Inserts or Updates

Say I'm performing a bulk insert of users

```php
$usersToInsert = getUsersFromCsv('/my-file.csv'); // 100 users
User::insert($usersToInsert);
/**
 * I have inserted 100 users in a single query, but no observers fire,
 * so they don't have Stripe accounts created.
 */
```

or I want to update all of the users belonging to a team

```php
$team->user()->update(['active' => false]);
/**
 * No observers fired, so we never soft-delete their invoices.
 */
```

These can be worked around, of course. Just create each user one by one, leading to 100 queries instead of 1 query. Or retrieve all of the team's users from the database and update them each in a separate query. But this means an additional strain on your database and a slower response time.

## Events

I think that the push for event-driven design gets a little muddy when talking about Laravel events. This can be because event-driven architecture in a lot of languages expects that the event is handled in a separate process, such as a greenlet thread. For instance, I dispatch an event and then an isolated thread of my application handles the event.  Or that events are written to a stream which are then handled by other processes (or even different downstream applications). But as we well know, PHP was designed to run in a single process: receive a request, do some application logic, and then return a response.

In Laravel, events are not queued, only their listeners can be queued. This allows us to dispatch an event (UserUpdated) and let some of the listeners run in the lifecycle of the request while others listeners handle the event later with a queue worker.

Events have all the same problems of obscurity, discoverability, cognitive load, and testing confidence that Observers do. The benefit often attributed to event-driven architecture is that we can dispatch an event and different domains can listen to the event without the tight coupling required for a single orchestrating command. This is absolutely true, but let's look at some of the other unique pain points.

### Order Of Operations

Does order of event handling matter to your application?  How do you guarantee the events are handled in a particular order?  What if someone comes along and makes a change to where event listeners are registered, swapping their order? How do you protect yourself against this?  If all of your event listeners are queued, there is no guarantee that the events will be picked up and worked in any particular order. For example, if an event listener that sends an email runs before the listener that updates an associated database record, you might end up with email content that is inconsistent.

### Exception Handling

If your events are not queued and are instead happening in the lifecycle of the request, any listener which throws an exception will break the application.  Is this desired?  Or do you want the response still returned to the user?  What if a failure in one listeners leaves the database in an invalid state, such as creating one row but not child rows in another table?

### Bad Citizens

Objects are passed by reference. Say your event looks like this:

```php
class UserUpdated
{
    use Dispatchable;

    public function __construct(public User $user) {}
}
```

It's possible that any listener which handles the event can modify the `$user` object, leading to failures in event listeners later in the process.

## What To Do Instead

So, I've given a lot of pain points, but how do we minimize them?

### Write Tests Which Ensure Application State

Fake your queue, notifications, and events by default. Always set expectations about events, notifications, and jobs that are dispatched.

### Rely On Actions

The [command pattern](https://refactoring.guru/design-patterns/command/php/example), often called actions in the Laravel ecosystem, are my preferred solution.  They are a class that is responsible for executing functionality and orchestrating other functions.  Cognitive load is reduced when we can see that: when I want to update a user, I call the `UpdateUserAction`, which will modify a row in the users table, and will then create modify their Stripe account and modify their existing invoices. Top to bottom, the functionality is in one file. I do not need to modify my tests because creating a user does not defer to the framework to handle out of process functions.

Additionally, decomposing functionality makes it easier to write testable components.  I can focus on a single function and test that it does what it is intended to do.  If I only test observers indirectly, such as by calling an endpoint which updates the user and confirming that the job was dispatched, my tests are very brittle and are tied to implementation details.  The more setup required to test something, the less likely I find myself actually testing every branch of that indirect function.

Here is an example of how we can test `UpdateUserAction` more directly.

```php
use App\Actions\UpdateUserAction;
use App\Models\User;
use App\Models\Invoice;
use App\Services\StripeService;
use Tests\Doubles\StripeServiceDummy;
use Tests\TestCase;

class UpdateUserActionTest extends TestCase
{
    // ... test that spies on the Stripe service to ensure we send the expected params ...

    public function test_invoked_updatesInvoices(): void
    {
        // Given
        $user = User::factory()->create(['business_name' => 'Old Name']);

        // And the User has invoices
        $invoices = Invoice::factory()->count(5)->for($user, 'user')->create(['sender' => 'Old Name']);

        // And we have faked a Stripe service bound to the container
        $this->app->bind(StripeService::class, StripeServiceDummy::class);

        // And we've updated our user's business name
        $user->business_name = 'My New Business Name';

        // When
        $this->app->make(UpdateUserAction::class)->__invoke($user);

        // Then
        $invoices->refresh();
        $this->assertCount(5, $invoices);
        $invoices->each(function($invoice) {
            $this->assertSame('My New Business Name', $invoice->sender);
        });
    }

    // ... more tests to confirm other functions of this action, like sending notifications
}
```

### Use Jobs and Batches

If functionality can be deferred, it probably should be.  Push that onto the queue. If you have a volatile dependency, like a third-party API, then jobs can be set to retry.  If you have a series of jobs that need to run, consider using a [job chain](https://laravel.com/docs/11.x/queues#job-chaining).

## Lessons Learned

Observers and events can lead to brittle tests, increased cognitive load, and unexpected behavior.  Consider these carefully when employing event-driven design in your growing Laravel project.

I can learn from another person, reading through the code of an open-source package, a blog post, the docs, et cetera, but it's important to think critically about its applicability to the project at hand, as well as the pitfalls.  New things can be alluring, but there is a cost associated with every architecture decision which may not be mentioned or obvious in a 20 minute conference talk.  The examples employed by blog posts to illustrate functionality are often devoid of things like testability, discoverability, and cognitive load.

It's important to remember that Observers and Events are not bad, but my experience has shown their benefits are outweighed by their cost.  Your constraints of team size, project scope, and code architecture are different than my constraints, so don't think this article is saying "Laravel model observers considered harmful" (though I was tempted to make that the title for bait :laughing:).

---

Have you had success with observers and events on a large team? How did you go about achieving that? Have you suffered pitfalls as the result of reliance on event-driven architecture in Laravel? Let me know on Twitter [@cosmastech](https://twitter.com/cosmastech).
