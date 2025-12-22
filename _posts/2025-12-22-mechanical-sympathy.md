---
title: "Mechanical Sympathy"
date: 2025-12-22
tags: active record, database, software engineering
---

Mechanical Sympathy was a term that I first learned from Matt Holiday's [excellent Golang course on YouTube](https://www.youtube.com/watch?v=7QLoOd9HinY).
I won't do it justice, but I might summarize it as _thinking about what a tool is doing under
the hood, and levaraging this knowledge to determine how you use it_. This applies to most technology,
from race cars (the origin of its meaning) to programming to how to load a dishwasher.

A dishwasher uses jets to shoot hot water and soap onto my plates and cups, then rinses them clean. The
water is forced out of a rotating arm on the bottom of the unit. If I stack my cups right side up, the
inside of the cups won't get sprayed as well, and when the cycle is done, I'll have cups filled with
water. If I recognize how the machine works, I know that placing the cups upside down in the top rack
of the unit is the best way to clean my coffee cup.

# My Reality (and likely yours too)

I work in backend web development, so most of what I am paid to write is CRUD apps. The data that my
code reads, writes, and modifies exists most often in a database and sometimes in external services
that I communicate with via HTTP requests. It's not glorious, but it pays my bills. And I love writing
code and solving problems.

Morgan Housel wrote about the human desire for complexity in his book Same As Ever, which immediately
struck a chord for me. You can read some of his thoughts about complexity in his blog post
[Why Complexity Sells](https://web.archive.org/web/20230609054524/https://collabfund.com/blog/why-complexity-sells/).
In short, complexity gives a sense of authority, mystique, and control. When what we do day to day
isn't warranting complexity, we have the tendency to make it complex to keep ourselves entertained or
create perceived value.

# Abstractions
