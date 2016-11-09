---
layout: post
title: "Why manage dependencies?"
date: 2016-11-09
categories: complexity
---

We're always told:
 - Make your code [loosely coupled](https://en.wikipedia.org/wiki/Loose_coupling)!
 - Use [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection)!
 - [Hide information](https://en.wikipedia.org/wiki/Information_hiding)!

But have you ever thought "why bother"? It nearly always slows you down, so why don't we just:
 - Keep classes coupled and just use the bits you require.
 - Make liberal use of singletons accessed directly in code.
 - Leave everything publicly accessible: [we're all consenting adults here](https://www.python.org/), amirite?


There are many reasons why we shouldn't do these things: I'm going to focus solely on the dependency aspects in this post.
 
## Code changes over time
 
First, you need to convince yourself that the demands placed upon the code you write **will** change over time. Requirements change. Management wants `$FEATURE` ready by yesterday. Startups "[pivot](https://en.wikipedia.org/wiki/Corporate_jargon)" to new ideas as the market changes. Assumptions you originally made when writing the code are no longer valid and your code needs to be changed to reflect reality. It happens. **It's inevitable.** No matter what the reason is, your next step will be to *change perfectly working code*. "If it ain't broke, don't fix it" isn't an option here.

## Changing code

To change code you need to:
 - Work out *what* needs to be changed in the code.
 - Find the exact code to change.
 - Change it.

Easy, right? Are we missing anything though? Were other parts of the program relying on some behaviour you have now changed? Enter: dependency management. **THIS** is why dependencies matter. You don't know what you may have broken when making this change. How do you find out what you may have broken? In most languages, you just find out what calls the function. You begin to form a **dependency graph** of:
 - Things that call this function.
 - Things that rely on the return value[s] from this function.
 
 
