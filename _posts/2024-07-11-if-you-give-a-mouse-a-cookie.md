---
title: "If you give a mouse a cookie..."
date: 2024-07-11
tags: Laravel, statsd, open-source, observability
description: How I created three packages to solve a single problem
---
As a child and now as a parent, I've always loved the children's book "[If You Give a Mouse a Cookie](https://www.amazon.com/Give-Mouse-Cookie-Paperback-Book/dp/B07T3VSVHK/)." Once you give the mouse his cookie, he asks for a glass of milk, and then he asks for a ... I refer to it often in software engineering discussions: once a door is opened (new feature is added, access is granted, et c), the effect turns into a cause for more change.

If you give a software developer an idea for something fun to build, he's probably going to want to develop three open-source packages in the process.

---

One of the most important lessons I learned in the past 3 months is the value of instrumentation within applications. Logs are great when you know there's a problem and want to see what's happening inside the application, but they do not provide quick feedback. They also can get very noisy. Metrics can tell us about the health of a system at a glance, and are much easier to trigger or alert on.

Some examples of useful metrics:

* How long does an external API call take to complete?
* How many users employ a certain feature?
* How many times is a product viewed in a shop?
* How many jobs are currently in a given queue?

I wanted to bring this valuable lesson to my work on an application written in Laravel. That should be simple, right?

## Some Requirements
I need to be able to easily write stats in a Laravel application. Something that I can bind to the container, and access via a Facade.

The first application I'll be working on will use DataDog for observability. DataDog has a first-party PHP library called [dogstatsd](https://github.com/DataDog/php-datadogstatsd). This is a superset of the [statsd](https://github.com/statsd/statsd) protocol, with some configuration options unique to DataDog.

But what if we want to write to a local statsd daemon while we do local development? The configuration for using the DataDog library wouldn't really work.

PHP League has their own [statsd client](https://github.com/thephpleague/statsd), which is widely used, but would not allow for writing some of the DataDog-only metrics ([like `histogram`](https://docs.datadoghq.com/metrics/types/?tab=histogram)).

It seemed like creating adapter class would do the trick. An adapter creates a unified interface and would allow for client code to swap out either DataDog or League's statsd package.

But what about unit testing?  I do not want to make having statsd running a requirement of the test suite, as the additional dependency takes more time to spin up and is another point of failure in CI pipelines.  While it's perfectly fine to mock classes, it often gets entangled with how the dependent classes are used. If we were to create a statsd spy that could plug into the adapter, then we could easily test if a stat is recorded on special conditions and exactly what is being passed.

What if some of the developers on my team don't want to run statsd at all?  One more point of failure, one more thing to add to my system start up. Wouldn't it be nice if I could just write them to a log instead?

## Let's Build
Oof... I'm not the only one who jumps into building stuff too quickly, right? I would love to say that I sat down, scratching my chin until I had the perfect application architecture... but this was not the case.

My first thought was "let's make this a package that's reusable because open-source is cool." So I created a folder named `statsd-laravel` and ran `composer init`.

As I started to work, building out an adapter interface and implementations for DataDog's and PHP League's clients, I realized that:

### Limiting this to Laravel reduces the visibility of the package.
Yes, I want to use it for Laravel, but the adapter itself isn't Laravel-specific. A Symfony app or a Slim app could certainly use it as well.

### There is low coupling
There's high cohesion: a Laravel facade and the implementation are used together but they don't need to update together. Upgrading to a new version of Laravel wouldn't change the adapter, but it could potentially change a service provider.

## Great, We'll Make Two Packages!
So I pivoted and decided to create a statsd-adapter package. This would contain everything needed to create the adapter class, but would have no dependency on Laravel.

I built an adapter interface, and concrete implementations for DataDog, League, and InMemory. The InMemory client designed for running in unit tests.

**But...** what about that concrete implementation that writes to a log file? DataDog has a slightly different way of writing to their statsd implementation, and I thought "it would be kind of neat to see what that looks like."

Unit testing writing to logs isn't necessary, we can just store them in memory. So in the statsd adapter package, I wrote a psr/log implementation to spy on what data is being written.  

**But...** isn't this the same problem I had before? This logger implementation could be useful when I (or others) build their own library and simple want to spy on the logs being written in unit tests.

## Ok, so three packages aren't THAT many
And we finally reached the inner layer of this 3 project package: [`cosmastech/psr-logger-spy`](https://github.com/cosmastech/psr-logger-spy). I imagine there are other implementations that do something similar. As of writing, [there are 155 psr/log-implementation packages on Packagist](https://packagist.org/providers/psr/log-implementation). I had already written my own, and it gave me experience releasing a package to Packagist.

In the end, the architecture follows a bit of an onion pattern.

![App Architecture](/assets/2024/laravel-statsd-architecture.png)

The StatsD client adapter relies on the Logger spy for unit testing. This allows for tests that just cover the adapter implementations.  Breaking the code into separate libraries allows for confidently testing each piece in isolation.

The laravel-statsd-adapter depends upon the adapter package for implementation. The tests within the Laravel package are very lightweight, as they only need to test exactly what's in the package, and not what its dependent libraries do.

## The final products
### cosmastech/laravel-statsd-adapter
Now you can [easily record statsd metrics in your Laravel application](https://github.com/cosmastech/laravel-statsd-adapter/). A handy `Stats` facade which allows for configuring multiple channels, as well easy dependency injection.

### cosmastech/statsd-client-adapter
A [statsd adapter package](https://github.com/cosmastech/php-statsd-client-adapter?tab=readme-ov-file) to be used in your PHP application. Check out the examples to see how this can work with different statsd clients.

### cosmastech/psr-logger-spy
This [logger spy](https://github.com/cosmastech/psr-logger-spy) may be useful in writing your own unit tests, or perhaps not.

---
This was my first foray into releasing an open-source package (or three). It was a great learning experience, a lot of fun to code, and wonderful practice. I'm excited to use it for work.  If you end up using this package, feel free to hit me on [Twitter](https://x.com/cosmastech), [LinkedIn](https://www.linkedin.com/in/luke-kuzmish-73a68b205/), or [GitHub](https://github.com/cosmastech).
