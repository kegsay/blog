---
layout: post
title:  "Programming Languages and Clarity"
date:   2017-01-05 18:00:17 +0000
categories: complexity
---

So much of what we do as programmers involves the need to express yourself clearly.
Whether it's writing code for others to read or writing documentation, if we cannot convey
the intent behind our writings, our program will not work as expected or our colleagues will
not be able to understand it. This post muses on how the choice of programming language can
affect overall program clarity, and how languages may change in the future to improve clarity.

### Programming language slider

When we write new projects, we need to decide which language to use. Sometimes this choice is
made for us: if we're writing for the web it needs to be JavaScript (or at least it used to
be: the number of transpilers to JS is growing!). Sometimes we stick with what we know and
have used before. Let's go with a hypothetical super programmer who knows all languages and
isn't forced to choose a particular language. Which language should they use? We need to be
able to compare programming languages to answer that.

First off, what things should we be measuring when weighing up different languages? Most
people settle on a combination of:
 - Performance.
 - Safety.
 - Prototyping speed.
 - Succinctness without being unclear.

Let's use Python as a guinea pig: it's fantastic to prototype in, you can express a lot
with very little code without being unclear, but it's really slow compared to compiled languages.

What about Java? It's very fast, but is reasonably verbose, and isn't great to
prototype in as it forces you to break things up into classes before you're ready to.

What about Go? Again, it's reasonably fast and is okay for both prototyping and succinctness.
The lack of decent [lodash](https://lodash.com/)-style utility libraries limits how expressive
you can be per line of code, and it isn't memory safe in the presence of data races thanks to the GC.

You may have noticed a pattern. There are tradeoffs: **there is no perfect programming language**.



### 
