---
title: "Simplicity Matters"
date: 2026-01-16
tags: software engineering
---

Simplicity matters. It's important for humans and our tiny brains. It's important for AI Agents with their tiny context windows. It's important for businesses that are fighting and clawing for market share as competitors clone their product in a span of months.

And simplicity is hard. It's hard because it takes experience to know what is essential and what makes it hard to understand. Seeing our code as a new developer might is nearly impossible, so when you get a new person working on your project, their insight can be priceless and unparalleled.

Simplicity also takes discipline. Morgan Housel, in his book [Same as Ever: Timeless Lessons on Risk, Opportunity and Living a Good Life](https://www.amazon.com/Same-Ever-Guide-Never-Changes/dp/0593332709), writes "Complexity gives a comforting impression of control, while simplicity is hard to distinguish from cluelessness." We write long docs because a few simple phrases makes us sound like we're not earning our keep. It's hard to admit that a lot of my work amounts to API endpoints and basic CRUD operations... I'm not writing video games or compilers. I've written complicated abstractions to keep my brain busy, to feel useful, to have something to do. I will tell you that it's highly likely I was unaware at the time, but I can see in retrospect that I introduced complexity.

Sticking with a codebase for even a little while, it has become apparent: complexity is a tax on change that is paid by new developers and veterans of the codebase. It's paid sprint after sprint, project after project. What could be a one or two line change in production code turns into a sprawling exercise in jumping through interfaces, adding new methods, modifying DTOs. Then come the tests... my god the tests that have to change.

## Inherent Complexity

A worthwhile software business sells a product that does something. If you're a saleable business, you're probably doing something interesting that has its own complexity baked in. I recently started a new job in a domain I have little experience in. Without even looking at code, there is time required just to understand what the business products do for its different types of users. Each tenat has a unique combination of settings and package add-ons, so their users and customers have an increasing number of experiences which our code must accommodate.

What's the opposite of inherent complexity? It's accidental complexity. It's the abstractions we build for their own sake, the dead-code graveyards that no one deleted, the unit tests that confirm a function is called through a chain of mocks instead of asserting against observable behavior, it's fighting against our tools.

## Our Tools

I am a little bit of a Laravel evangelist, it seems. Who knew? I have worked with Laravel for around 5 years, and I have come to intimately understand the framework and its internals. The selling point of a batteries-included, rapid application development web framework is you can get something that just works in less time than it takes to roll your own. It's not reasonable to expect everyone who drives a car to be able to rebuild an engine or even explain how it works. Similarly, it's not reasonable to expect everyone who uses Laravel to study the release notes to see which features were added to the framework, often times a dozen or more in a week.

For those who do keep apprised of changes to their tools, they can see the tool often evolves to meet needs of other folks. When the tools change, oftentimes it is to solve a problem that I may have too. One of the secrets about why I contribute so frequently to Laravel is because I have problems that I want solved at the first-party level. I don't want to have to build hacky workarounds or extend and override classes to get the functionality I need. Let Taylor handle the maintenance. :laugh:

### Use the Magic, but Understand the Magic

Frameworks like Laravel, Rails, and Django can be disliked. Sometimes for the opinions then strongly enforce, or because they have too much magic. I cannot say that Laravel doesn't have lots of magic. I'm a curious person and I don't like magicians because I just want to know how they do the trick. I'm also a software developer who can read PHP and has a step debugger; I can see exactly how the magician does the trick. None of us is too busy to get a baseline understanding of how our tools work.

There are big payoffs for understanding (at even a basic level) how things like queued jobs work end-to-end; models and relationships execute queries; and how the Container magically slots in your dependencies. Knowing little bits of the magic gave me that feeling of being in control, the same feeling of control I get if I were to roll it all by hand.

### But You Don't Have To Use All the Magic

There are plenty of things that I won't use in Laravel because I recognize they don't fit my use case. Hey, [have I mentioned that I don't like model events and observers](https://cosmastech.com/2024/08/18/laravel-observers-and-models.html)?  They cause indirection, don't fire when inserting records in bulk, and are hard to discover. So I don't use them for projects I work on.

But it's important to note that while some things are difficult to use at scale work great for a single developer who has the entire project in their head. I encourage you to consider your working environment with an open-mind, doing your best to not be emotionally attached to the code you have written.

For those who are interested, I like Models to function as:

* a simple data holder with appropriate casts and maybe some "ask me about" methods (`canReceiveStripePayments()`)
* a way to query for related Models via relationships
* an excellent query builder with convenient scopes
* a novel way to think about my data access patterns
* building blocks of integration tests through Factories

I don't put a lot of functional logic in them. I do not feel these Models don't suffer from anemia, they thrive.


## Dogma versus Pragmatism

I have read Clean Code. I have implemented Clean Code. I have been sad.

Holding fast to rules in the face of a reality to the contrary is a recipe for complexity without payoff. We must be like trees in the wind, bending don't so we don't snap. I had a mentor early on who, after me asking what the rule is for something, would tell me "you have to use your brain." Being realistic about the size of a project or the makeup of my team bears better fruit than any book or blog post.

For instance, repositories. I have written them. They end up looking like this:

```php
// Use the suffix -Entity because I know the technical distinction between an entity and a value object
// and it's VERY important you know that I know. (This is a true story.)
final readonly class UserEntity
{
    public function __construct(
        // private because TECHNICALLY no other developer should be allowed to know our secret integer ID
        private int $id,
        public string $uuid,
        public string $firstName,
        public string $lastName,
        public EmailAddress $emailAddress,
    ) { }
    
    public function updateEmail(EmailAddress $email): UserEntity
    {
        // this probably does something great, but everything must be immutable,
        // so I'll definitely want to clone this object and return a new one
    }
    
    public function flush(): array
    {
        // Outbox pattern mentioned!
    }
}

final readonly class UserRepository implements UserRepositoryInterface
{
    public function get(string $uuid): ?UserEntity
    {
        return User::query()->firstWhere('uuid', $uuid)?->toEntity();
    }
}
```

They end up wrapping Eloquent functionality, but with worse developer experience. Each time I need to do a slightly different query, I end up writing a new repository method. If I add a property to a core entity, trust and believe, I'm now going to have to update `UserEntity`, `UserMapper`, `UserRepository@update()` and `UserRepository@create()`, and probably a bunch of other places.

...oh and all of those unit tests I wrote! :sweat:

Is Eloquent always right for the job? No. The questions to ask are: how does this complexity serve us? Does it make our business more attractive to clients? Does it allow us to ship features faster? Does it make it easier to change our code now and in the future? Is it easier for an LLM to parse and generate code for? Does it make it easier to debug? What are we protecting ourselves from? What would happen if we made it easier?
