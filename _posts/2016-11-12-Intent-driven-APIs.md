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

In essence, intent-driven API design is designing APIs to work in a way which meets the intention
of the user. This sounds obvious, so let me explain with some examples. I'm going to pick on the
[WebRTC API](https://w3c.github.io/webrtc-pc/#simple-peer-to-peer-example) for browsers because I've used
it in the past and it neatly illustrates the idea. This API allows audio and video to flow directly between
two browsers. It's primarily used for video calling directly from a browser.
Here is the "simple peer-to-peer example":

```js
var signalingChannel = new SignalingChannel();
var configuration = { "iceServers": [{ "urls": "stuns:stun.example.org" }] };
var pc;

// call start() to initiate
function start() {
    pc = new RTCPeerConnection(configuration);

    // send any ice candidates to the other peer
    pc.onicecandidate = function (evt) {
        signalingChannel.send(JSON.stringify({ "candidate": evt.candidate }));
    };

    // let the "negotiationneeded" event trigger offer generation
    pc.onnegotiationneeded = function () {
        pc.createOffer().then(function (offer) {
            return pc.setLocalDescription(offer);
        })
        .then(function () {
            // send the offer to the other peer
            signalingChannel.send(JSON.stringify({ "desc": pc.localDescription }));
        })
        .catch(logError);
    };

    // once remote video track arrives, show it in the remote video element
    pc.ontrack = function (evt) {
        if (evt.track.kind === "video")
          remoteView.srcObject = evt.streams[0];
    };

    // get a local stream, show it in a self-view and add it to be sent
    navigator.mediaDevices.getUserMedia({ "audio": true, "video": true })
        .then(function (stream) {
            selfView.srcObject = stream;
            pc.addTrack(stream.getAudioTracks()[0], stream);
            pc.addTrack(stream.getVideoTracks()[0], stream);
        })
        .catch(logError);
}

signalingChannel.onmessage = function (evt) {
    if (!pc)
        start();

    var message = JSON.parse(evt.data);
    if (message.desc) {
        var desc = message.desc;

        // if we get an offer, we need to reply with an answer
        if (desc.type == "offer") {
            pc.setRemoteDescription(desc).then(function () {
                return pc.createAnswer();
            })
            .then(function (answer) {
                return pc.setLocalDescription(answer);
            })
            .then(function () {
                var str = JSON.stringify({ "desc": pc.localDescription });
                signalingChannel.send(str);
            })
            .catch(logError);
        } else if (desc.type == "answer") {
            pc.setRemoteDescription(desc).catch(logError);
        } else {
            log("Unsupported SDP type. Your code may differ here.");
        }
    } else
        pc.addIceCandidate(message.candidate).catch(logError);
};

function logError(error) {
    log(error.name + ": " + error.message);
}
```

Now WebRTC has a *lot* to do. It needs ICE candidates, exchanging offer/answers to negotiate which
codecs to use, lots of things which are very complicated and very much *required* in order to set
up a call. But what is the intent of the average user of this API? What are they thinking when they
use this API? **What does the developer using this API want to actually do?** For WebRTC, 90% of the
time the answer will be setting up a video/audio call. That's it. What does that entail? It requires:
 - Ability to talk to a remote IP address.
 - Ability to receive incoming calls.
 - Ability to answer/reject a call.
 - Ability to hang up an active call.

When a developer decides to use the WebRTC API, they will need to work out how to do these things. They
may not know the WebRTC stack. They may not know STUN/ICE/RTC/SRTP/$acronym, they *just* want to make a call.
API designers should strive to make their APIs easy to use. We know about abstraction. We know the idea
that "outsiders" shouldn't need to know or care about X, Y or Z and yet we still design APIs which leak
these details like a sieve. Here's what [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/WebRTC_Basics#Answering_a_call)
says you need to do in order to answer a call:

```js
var offer = getOfferFromFriend();
navigator.getUserMedia({video: true}, function(stream) {
  pc.onaddstream = e => video.src = URL.createObjectURL(e.stream);
  pc.addStream(stream);

  pc.setRemoteDescription(new RTCSessionDescription(offer), function() {
    pc.createAnswer(function(answer) {
      pc.setLocalDescription(answer, function() {
        // send the answer to a server to be forwarded back to the caller (you)
      }, error);
    }, error);
  }, error);
});
```

Now if you know WebRTC, you may be thinking "how does it get any simpler than this?". If you don't know
WebRTC, you may be thinking "what's user media?" or "what's a session description"?

## Conclusion
