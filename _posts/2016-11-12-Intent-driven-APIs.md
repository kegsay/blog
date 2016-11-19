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
that the intent of the user is often very different from the actual implementation details, and
therein lies the problem.

### Example: VoIP
Let's say we're making a VoIP API. We can place calls, receive calls, answer/reject calls and hangup.
These operate on underlying "data streams" which need to be set up and torn down when the call is
answered/hung up, and it might take some time to do both of those operations. What might the hangup
function look like? *The following examples assumes knowledge of coroutines / async functions*.

```js
class Call {
    hangup() {
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
            throw new Error("Already answering");
        }
        this._answering = true;
        // assume setup cannot fail
        this.dataStream = await this._setupDataStream();
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
right times. This is awful, but a lot of APIs do this. How else could this have been done?

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

```js
async onAnswerClick() {
    await call.answer();
    // change the UI
}

async onHangupClick() {
    await call.hangup();
    // change the UI
}
```

That's a lot better! They no longer need to remember what state the call is in, and it greatly simplifies their code.

### Example: Instant Messaging
Let's say we're making an IM API. We have the concept of "rooms". Members need to inside a "room" before they can speak,
and their message goes to every member of the room. Members may have different privilege levels which prevent them from
speaking. What does the speak function look like? *These examples omits all kinds of errors and asynchronous concerns
for simplicity*.

```js
class Room {
    speak(text) {
        if (!this.isInsideRoom() || !this.hasPermissionToSpeak()) {
            throw new Error("You cannot speak in this room.");
        }
        // send message to the room
    }
}
```

How might a developer use this function? Perhaps when they hit a "SEND" button:

```js
onSendClick() {
    let text = this.getTextFromInputBox();
    room.speak(text);
}
```

But this will throw an error if they aren't inside the room or have permission to speak. So they need to check this first:

```js
onSendClick() {
    if (!room.isInsideRoom()) {
        room.enter(); // assume this can never fail and is synchronous
    }
    if (!room.hasPermissionToSpeak()) {
        room.requestPermissionToSpeak(); // assume this can never fail and is synchronous
    }
    let text = this.getTextFromInputBox();
    room.speak(text);
}
```

Notice the pattern? They're performing the exact same checks as `speak(text)` because the API **is forcing them to**. This
is just a very simple example. It can quickly spiral out of control. Perhaps you need to already be in the room before you
can request permission to speak (so ordering now becomes important). The user of this API now has to know a great deal about
the internal state of `Room` in order to actually use it. How might this be improved? Helper methods are the key here:

```js
class Room {
    forceSpeak(text) {
        if (!room.isInsideRoom()) {
            room.enter();
        }
        if (!room.hasPermissionToSpeak()) {
            room.requestPermissionToSpeak();
        }
        room.speak(text);
    }
}
```

This means that the naive `onSendClick` we had at the beginning would work with this function:

```js
onSendClick() {
    let text = this.getTextFromInputBox();
    room.forceSpeak(text);
}
```

## Conclusion
