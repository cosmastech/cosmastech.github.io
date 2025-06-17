---
title: "FailOnException: Short-circuit Laravel job retries"
date: 2025-06-17
tags: [Laravel, PHP, queues]
description: A simple way to stop a job when further attempts will fail.
---

## The Problem
We've all had this problem. We have a queued job that grabs some data from the database, performs some checks on it, and then inserts a new record or calls out to an external API.
We want the job to retry if the API request fails or there's a hiccup with writing to the database, however, if any of the checks fail, we want to mark the job as failed and not
bother retrying it.

Not seeing anything obvious in the Laravel docs, we reach for Claude to tell us how to do this. It hallucinates some methods that don't exist, and now we have our original problem
and we're a little pissed off with AI.

**We've all been there.**

But with the release of [Laravel 12.19](https://github.com/laravel/framework/releases/tag/v12.19.0), there's an easier way.

## `FailOnException` Job Middleware

