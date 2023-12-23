---
title: "What I have been working on in November 2023"
date: 2023-11-26
tags: open-source, Golang, python
description: How I'm learning Golang and Python by contributing to open-source projects.
---

## A bit about my background

The primary language for my professional software development career has been PHP. It wasn't a choice I made: I needed money and the easiest-to-grab jobs in freelancing at the time
were working with PHP. It was Christmas Day 2014(?) that I took my first job as a freelance software developer.
The particulars are unclear, but I do recall it was from [/r/forhire](https://www.reddit.com/r/forhire/) and it was some kind of simple scraper that the client wanted written in PHP. *Yes, you can write web scrapers in PHP.* 

I kept taking PHP jobs, with a little bit of node and frontend work thrown in. Thankfully after man rough and tumble years of writing hideous vanilla PHP, I found Laravel and later started working on a team writing PHP with Laravel. From there, I grew as a developer by degrees.

## The present
I got an itch this month to start learning a new language. I think the _why_ is probably even a bit unclear to me, but I felt like I should be investing in my knowledge portfolio.

I personally find to-do apps, math problems solved with code, and most tutorials to be incredibly boring.  I thought a better way to learn might be to search for GitHub issues labeled "Good first issue" and see if I could understand the ask and submit a PR to resolve the issue.

I have felt this is really fun, though I have to remind myself that just because something is a _good first issue_ does not mean it's a good issue for someone with 5 hours of experience in the language.


### Golang
Go is a pretty slick little language. It has a syntax that reminds me of Python, static types (with type inference), and it compiles quickly.

According to [devjobsscanner](https://www.devjobsscanner.com/blog/top-8-most-demanded-programming-languages/), go is the 8th most in-demand language.

#### Resources
* [The Little Go Book](https://www.openmymind.net/assets/go/go.pdf) by Karl Seguin. I found this to be incredibly readable and a nice resource to quickly reference to understand or refresh concepts.
* [golings](https://github.com/mauricioabreu/golings) created by Mauricio Antunes. Inspired by rustlings for learning rust, this was a nice little way to practice basic Go skills.

#### Contributions
* [golings: fix typos](https://github.com/mauricioabreu/golings/pull/15) - In the true spirit of open source, I wanted to give back to something that helped me.
* [json-log-viewer: support human readable timestamps](https://github.com/hedhyw/json-log-viewer/pull/38) - this is a CLI tool for viewing JSON logs. The creator, [Maksym Kryvchun](https://github.com/hedhyw), was incredibly patient guiding me on my first Go contribution. I learned about Go's standard library `time` package and how to write unit tests. I think most striking about writing unit tests is the ability to parallelize Go tests.
* [FerretDB: Add option to disable --debug-addr](https://github.com/FerretDB/FerretDB/pull/3698) - FerretDB is a replacement for MongoDB that uses PostgreSQL or SQLite to back the data persistence layer. My contribution was a minor one: to disable writing to a debug location if a CLI flag was empty.
* [sbctl: create-keys allows for specifying an export directory](https://github.com/Foxboron/sbctl/pull/259) - SBCTL is a boot manager that targets Arch Linux. My contribution was to add the ability to specify export paths. I got to play with the Cobra CLI library a bit here.

#### Thoughts
I really like Go. I think the biggest challenge for me was finding projects that I could truly understand **why** they existed. Go is incredibly popular in the world of devops and containers, which I don't have a great deal of exposure to.

I would like to continue to work with Go and a find a place where it dovetails with my needs or the needs of the team I work on.


### Python
I've worked with Python before, even on a professional level (I once found a job on Elance to write a student's computer science assignments in Python 😆)
But I have been pretty averse to it. Python however is one of the most in-demand programming languages and is the glue language for AI/ML. I figured that if there
was a place to invest time into growing my knowledge, Python would probably pay the greatest dividends.

#### Contributions
* [boaviztapi - Uncaught exception to include CORS middleware](https://github.com/Boavizta/boaviztapi/pull/243) - FastAPI is an incredibly popular web framework, so I wanted to find a project that had an open issue which utilized FastAPI. If an uncaught exception bubbles up, middleware is not applied to the response. In this case, there's no CORS headers added to the response, so the 500 response won't necessarily be returned to the end-user. This was fun to work on and pretty informative. I learned about exception handling and that context can be suppressed (which Python adds when an Exception is raised while handling another Exception)
* [talkdai/dialog - Allow setting model_name in the prompt's TOML](https://github.com/talkdai/dialog/pull/51/files) - One of the things about Python that has kind of confused and frustrated me is the declaration of variables which are importable in other packages. I'm very used to the namespacing of PSR standard PHP, so this always makes me scratch my head a bit. I also learned that a Python dict has a handy method for getting a value by key OR returning a default.
* [cogent3 - Filter available_apps by name](https://github.com/cogent3/cogent3/pull/1637/files) - I love that Python now has typehints. My understanding is that typehints are more a nicety than a true enforcement of types, which I dislike. I find the `{str} in {another_str}` format to be a big departure, but something I imagine I could get used to.
* [monkeypatch/tanuki.py - Telemetry can be disabled](https://github.com/monkeypatch/tanuki.py/pull/90) - class level static variables are unintuitive to me coming from PHP-land, so realizing I had to reference the class by name within a static method was a learning opportunity.

#### Thoughts
Python requires a good IDE due to its dynamic typing. And even using [PyCharm](https://www.jetbrains.com/pycharm/), I found that it was often a struggle to find which function was being called on an object. I hope that as typehints become more widely adopted, this problem will lessen.

Poetry, pyenv, venv... wtf? There's too much going on here, and I struggled with each project to get the environment set up. PyCharm does help with that, to some degree, but it's still not perfect. It really helps me appreciate [composer](https://getcomposer.org/), PHP's package manager as well as tools like Laravel's [valet](https://laravel.com/docs/10.x/valet).


## Final Thoughts
Open-source maintainers of smaller projects have been so incredibly kind to me in my journey.
The majority of my open-source contributions have been to Laravel and its associated packages, where the environment is a lot different.
There is a lot less patience for questions or people who want to just learn. This is not an indictment of that community, it's the nature of a large-scale project.

I think I will continue to focus on Python for a bit. Given the number of popular uses of the language, it presents opportunities to go broad before going deep.
