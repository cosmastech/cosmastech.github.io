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

I am a little bit of a Laravel evangelist, it seems. Who knew? I have worked with Laravel for around 5 years, and I have come to intimately understand the framework and its internals. The selling point of a batteries-included, rapid application development web framework is you can get something that just works in less time than it takes to roll your own. It's not reasonable to expect everyone who drives a car to be able to rebuild an engine or even explain how it works. Similarly, it's not reasonable to expect everyone who uses Laravel to know about each feature that gets added to the framework, often times a dozen or more in a week.
