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
 - Keep classes tightly coupled and just use the bits you require. You probably aren't ever going to need the class elsewhere.
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

Easy, right? Are we missing anything though? Were other parts of the program relying on some behaviour you have now changed? Enter: dependency management. **THIS** is why dependencies matter. You don't know what you may have broken when making this change. How do you find out what you may have broken? In most languages, you just find out what calls the function. You begin to form a **dependency graph**<sup>[1](#deps)</sup> of:
 - Things that call this function.
 - Things that rely on the return value[s] from this function.
 
But this is easy: IDEs often have "Find Usage" functions and there's always grep, right<sup>[2](#reflection)</sup>? But what happens if you actually find out you have broken something? You MUST fix it **before** you can land `$FEATURE`, or else it won't work. So you do the same 3 steps:
 - Work out how this piece of code was using the function before. Work out *what* needs to be changed.
 - Find the code to fix.
 - Change it.

Rinse and repeat. On bad codebases this can end up with many disparate code alterations to add in a single change/feature/bugfix. On really bad codebases it may turn out to be **impossible** to realistically make the desired code alteration because it would require completely re-architecting the project. The project ends up [gradually petrified](http://finalfantasy.wikia.com/wiki/Gradual_Petrify_(status)) and there is no realistic way out of it. How could this have been avoided?


<a name="deps">1</a>: *It's important to point out that the dependency graph you form when modifying code transcends any kind of "API" you may have cobbled together. It doesn't matter what the public API is if you gut wrench private `_variables` in Python for example: you're depending on it.*

<a name="reflection">2</a>: *This works up to a point: if your project makes liberal use of reflection/metaprogramming then these aids will not help you. What's more, your compiler (if you have one) probably won't help you either, obscuring your dependency graph.*

## Minimise dependencies

The problem we encountered when changing code was that we kept having to "hop" from one section to another to follow the dependency graph. An easy solution would be to not have code depend on other code, but that simply isn't realistic. There's always dependencies: *something* must be using the code you have written else it shouldn't exist in the first place. If you accept that, then there will always be a dependency graph: **but you can shape how it looks**. In order to do that, you need to have a clear idea of what it looks like. This is very hard to do when all your dependencies are implicit (e.g. accessing singletons directly in functions). This is where the idea of "loosely coupled" components and dependency injection come from. They help to reveal the hidden world of dependencies in your code, so when you want to change it you can do so more easily.

Let me take this moment to emphasise that you can [take this to the extreme](https://github.com/EnterpriseQualityCoding/FizzBuzzEnterpriseEdition). I **strongly** disagree with complex [DI frameworks](https://projects.spring.io/spring-framework/) that do nothing but obscure your code: [there are clearer alternatives](https://en.wikipedia.org/wiki/Dependency_injection#Constructor_injection) which work in all languages.

If you search for it, you soon realise dependency graphs are everywhere but we don't know what they look like or how deep they go. You find them in your package managers, your OS, your LAN and your HTTP APIs. You're literally drowning in implicit and explicit dependencies. But this is the nature of software development. Provided everything works, complex dependency graphs are fine. It's only when something [breaks](http://www.theregister.co.uk/2016/03/23/npm_left_pad_chaos/) do we realise how big the house of cards really is.

So consider what your dependencies are. Make sure they are clear. There is no silver bullet.
