---
title: "Improving unit test runtime in Laravel modular monolith"
date: 2025-11-13
tags: Laravel, Testing, modular, PHPUnit
---

I recently started a new job and was given my first exposure to a [modular monolith](https://laracasts.com/series/modular-laravel). On the surface, they have a lot
to love: all aspects of your app in one repo, one set of dependencies to keep updated, and code duplication are first that spring to mind. They also of course
present some challenges, and the one that kind of struck me by surprise was test suite run time.

In previous projects, test suites usually took only two to three minutes to run locally. Parallelization obviously helps a great deal, but it is not a silver
bullet. I have spent a great deal of my focus on writing tests that don't execute extraneous database queries:
* don't build more than you need
* leverage bulk inserts versus many inserts in a loop
* be mindful of model events that may trigger database queries
* test a single unit of code when possible
* leverage `Factory::make()` if writing to the database isn't strictly necessary

These are helpful guidelines, but an unfamiliar codebase with 8000+ tests is not something that can be optimized quickly.

## Wait, it takes how long?
In our pipeline, just the PHPUnit tests were taking between 8 and 10 minutes to execute fully. Heaven help you if you needed something merged quickly. The
unit tests were going to be the gate. This mean we were running about 800 to 1000 tests per minute. Not slow, but long enough. Any process for me as a dev
that takes long enough for me to think "Oh, I'll just do <xyz> while I wait" leads to me losing context, and usually forgetting what I was waiting on in
the first place.

## The Bottleneck
I had theories. Slow database queries, I figured. A staff engineer reached out to ask me if I saw anything obvious that would be slowing down our application
boot times. He had made the effort to convert some of our feature tests to plain old PHPUnit tests and saw major improvements in the speed. He's a smart
guy and I trusted his intuition, so I decided to go down a rabbit hole.

Leveraging [Herd's profiler](https://herd.laravel.com/docs/macos/debugging/profiler#profiling-cli-scripts), I decided to take a look at where our time
and memory consumption was getting eaten up. What I found was a bit surprising.

