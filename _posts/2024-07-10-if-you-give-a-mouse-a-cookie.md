---
title: "If you give a mouse a cookie..."
date: 2024-07-10
tags: Laravel, statsd, open-source, Reading
description: How I created three packages to solve a single problem
---


One of the most important lessons I learned in the past 3 months is the value of instrumentation within applications. Logs are great when you know there's a problem and want to see what's happening inside the application, but they do not provide quick feedback. They also can get very noisy. Metrics can tell us about the health of a system at a glance, and are much easier to trigger or alert on.

Some examples of useful metrics might be:
* how long do certain external API calls take to complete?
* how many users employ a certain feature?
* how many jobs are currently in a given queue?

I wanted to bring this valuable lesson to my work on an application written in Laravel. That should be simple, right?

## Some requirements
The first application I'll be working on will use DataDog for observability. DataDog has a first-party PHP library called [dogstatsd](https://github.com/DataDog/php-datadogstatsd). This is a superset of the [statsd](https://github.com/statsd/statsd) protocol, with some configuration options unique to DataDog.

But what if we want to write to a local statsd daemon while we do local development? The configuration for using the DataDog library wouldn't really work.

PHP League has their own [statsd client](https://github.com/thephpleague/statsd), which is widely used, but would not allow for writing some of the DataDog only metrics ([like `histogram`](https://docs.datadoghq.com/metrics/types/?tab=histogram)).

It seemed like an adapter class would do the trick. An adapter creates a unified interface and would allow for client code to swap out either DataDog or League's statsd package.

But what about unit testing?  I do not want to make having statsd running a requirement of the test suite, as the additional dependency takes more time to spin up and is another point of failure.  While it's perfectly fine to mock classes, it tends to get tied up with how the class is used. If we were to create a statsd spy that could plug into the adapter, then we could easily test if a stat is recorded on special conditions and exactly what is being passed.

Finally, what if some of the developers on my team don't wish to have statsd running at all?  One more point of failure, one more thing that I forget to 