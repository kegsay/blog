---
layout: post
title:  "Intent-driven APIs"
date:   2016-11-12 18:00:17 +0000
categories: complexity
---

API design is Hard. There are hundreds of decisions you need to make when designing a new API.
There's a long list of software design philosophies which can help you make these decisions such as:
 - The principle of [least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).
 - Having [sensible defaults](https://en.wikipedia.org/wiki/Convention_over_configuration).
 - [KISS](https://en.wikipedia.org/wiki/KISS_principle)

The list goes on. Even the [Zen of Python](https://en.wikipedia.org/wiki/Zen_of_Python) are good guiding
principles when designing a new API, be it an HTTP API or an API for a specific programming language. I've
read an awful lot about API design, and made my fair share of mistakes designing APIs. I've learnt a lot in
the process. One thing that I wish I was told, and yet very rarely see on programming blogs/books, is the
idea of **intent-driven** API design. That is what this post is about.

## What is Intent-Driven API design

In essence, intent-driven API design involves writing APIs to work in a way which meets the intention
of the user. This sounds obvious, so let me explain with some fictitious examples. What we'll notice is
that this intent is often far away from reality, and therein lies the problem.

### Example: VoIP
Let's say we're making a VoIP API. We can place calls, receive calls, answer/reject calls and hangup.
These operate on underlying "data streams" which need to be set up and torn down when the call is
answered/hung up, and it might take some time to do both of those operations. What might the hangup
function look like? *The following examples assumes knowledge of coroutines / async functions*.

```js
class Call {
    hangup(): void {
        if (this.dataStream) {
            this.dataStream.tearDown();
        }
    }
}
```

Aha, but `tearDown` might take some time. We probably want to tell the user when they have successfully hung up, right?
So let's fix that up:

```js
class Call {
    async hangup() {
        if (this.dataStream) {
            await this.dataStream.tearDown();
        }
    }
}
```

Great! Now what might an actual developer *do* with this function? Well, they might call `hangup()` in response to a
button click on the UI, and change the UI when the call has finished:

```js
onHangupClick() {
    await call.hangup();
    // change the UI
}
```

Now comes the fun part. What does the `answer` function look like? We need to be careful not to set the stream up
twice. Perhaps:

```js
class Call {
    constructor() {
        this._answering = false;
    }
    
    async answer() {
        if (this._answering) {
            throw new Error("Already answering"));
        }
        this._answering = true;
        // assume setup cannot fail
        let dataStream = await this._setupDataStream();
        this.dataStream = dataStream;
        this._answering = false;
    }
}
```

We have just created a racey API. What happens if the user does:

```js
let call = new Call();
call.answer();
call.hangup();
```

The `hangup()` code will do nothing because there is no data stream yet. This is a fact of life.
The streams take time to set up and tear down, and the API is just telling the user the whole truth and nothing
but the truth. The problem is that most of the time the user of the API doesn't care about the truth. They just
want to hangup the call. This is where you have a choice: Do you accomodate their whim and gloss over the fact
that these things take time, or do you tell them to [RTFM](https://en.wikipedia.org/wiki/RTFM) because this is
how it is. If we do **NOT** accomodate their whim, what happens? They have this bug they want to fix. How can
they fix this? Perhaps they should wait until the answer completes first:

```js
let answerPromise = null;

async onAnswerClick() {
    answerPromise = call.answer();
    await answerPromise;
    answerPromise = null;
    // change the UI
}

async onHangupClick() {
    if (answerPromise) {
        await answerPromise;
    }
    await call.hangup();
    // change the UI
}
```

Now you might have noticed a pattern here: `answerPromise` and `Call._answering` are *the same thing*. The user
of the API is effectively **copying over the state machine** of `Call` so they can call the right methods at the
right times. This is awful, but a lot of APIs do this. How else could this be done?

Let's revisit our choice and instead decide to accomodate their whim. What does `Call` naturally look like now?

```js
class Call {
    constructor() {
        this._answerPromise = null;
    }
    
    async answer() {
        // if not answering a call
        if (!this._answerPromise) {
            this._answerPromise = this._setupDataStream();
        }
        this.dataStream = await this._answerPromise;
        this._answerPromise = null;
    }
    
    async hangup() {
        let dataStream = this.dataStream;
        // if answering a call
        if (!dataStream && this._answerPromise) {
            dataStream = await this._answerPromise;
        }
        if (!dataStream) {
            return;
        }
        await dataStream.tearDown();
    }
}
```

We've shifted the complexity to the `Call` object now, so what does the user's code look like now?


## Conclusion
