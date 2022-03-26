---
title: "Adventures with the Streaming API"
description: "A brief story about how not reading the docs correctly can cost you some extra development time."
date: 2022-03-26T20:25:44+08:00
lastmod: 2022-03-26T20:25:44+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["javascript", "fetch", "streaming-api"]
categories: ["JavaScript"]
series: []

toc:
  enable: no
---

This story relates to [The Streams API in JavaScript and Go]({{< ref "/posts/the-streams-api-in-javascript-and-go" >}}) which I wrote a while back.

To add some context, I was working on a simple app, which is similar to a router dashboard, for one of the products the company I work ([Transcelestial](https://transcelestial.com/)) for is developing. And one of the main features of it was realtime metrics from the device.

As described in the prev post, I could have went with either [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) or the [Streams API](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API) (long polling was not an option because we deal with modern browsers), but I chose to go with the streams API, because there was no need for bidirectional comm b/t the client and the server and all the APIs were already in place, so it was easier to just add the extra handler to stream the data.

So everything went great and smooth. The streaming endpoints were running w/o issues for quite some time. Well, that's what I thought.

While developing this web app, I occasionally saw some error in the console:
```text
SyntaxError: Unexpected token { in JSON at position 119
    at streamData (<anonymous>:20:40)
```

But it happened so randomly and seldom that I just ignored it for quite some time. It also only seemed to happen after leaving the app running for quite some time (at that point it seemd like hours - but it turned out to be around 25 minutes pretty consistently).

So time went by and the day of field testing got closer and closer (when we actually test our devices in the field). So did my anxiety :laughing: . But I was not worried at all about this and I actually forgot about it. But then I started seeing the issue more often and during one of the field tests I noticed some states were not updating :exploding_head: .

Ok. So about this time is when I embarked on a journey of self doubt and madness.

My first thought was *"something must be wrong with the streaming code"* (I've taken the shared example in the prev post and generalized it so I can reuse for multiple endpoints). So I spent a few hours going through the unit tests and adding more of them until I realized that the issue wasn't there.

So if the issue wasn't there, then it must be the client. So I spent some more time going through that code (no unit tests :disappointed: ) just to end up with nothing :grimacing: . To add to the dismay, I could only reproduce the issue after waiting for about 25 - 30 minutes. So the dev cycle at that point was: make some changes in the API, make some changes in the client and wait for the error to happen, then look at the logs and try to figure it out; then repeat.

At that point, it's probably been about a couple of days and I had no answer. So I realized I'll have to open some issues somewhere to report this and maybe get some help. My thinking was *"it must be either in the echo library, the net/http/httputil or net/http libs"*. I was hopeless :weary_face:

So I started taking the parts responsible for streaming and setting up a simple echo server and started testing in the browser. But the environment I was dealing with was a bit more convoluted/complex:
1. The API was running on an embedded device (somewhere remote)
2. The API was cross-compiled
3. The API is served over HTTPS (HTTP/2) using self signed certs
3. There was a reverse proxy on top of the API (this was to address a different issue we had with self signed certs and how browsers deal with them)
3. The client was running on my machine
4. There was a VPN setup b/t my machine and the device so that I can develop the app (I know, I could have mocked the APIs, but that's another problem for later)

So I went for the most obvious scenarios to try to reproduce the issue in isolation:
1. An [echo server](https://echo.labstack.com/) <> an [echo reverse proxy middleware](https://echo.labstack.com/middleware/proxy/) <> a browser client (a simple js func to read the stream and parse to JSON) - closer to my env
2. An echo server <> a [net/http/httputil](https://pkg.go.dev/net/http/httputil) reverse proxy <> a browser client - to eliminate the echo middleware as an issue
3. A `net/http` <> a `net/http/httputil` reverse proxy <> a browser client - to remove the echo framework as having the issue
4. A `net/http` <> a browser client - to remove the reverse proxy as an issue
5. All of the above but with a Go client instead of a browser one

At the same time as I was adding the above scenarios to a repo, I started opening issues:
1. [labstack/echo#2125](https://github.com/labstack/echo/issues/2125)
2. [golang/go#51646](https://github.com/golang/go/issues/51646)

So as you can see, I was pretty convinced the issue was on the backend. But as I started testing more scenarios, I also added other browsers to the mix (so far I've only been testing in Chrome). This is when things got dark.

1. Firefox was not failing around the same time as Chrome (Chrome at the 25 min mark, Firefox had inconsistent times)
2. Firefox was also giving me different errors
```text
SyntaxError: JSON.parse: unexpected non-whitespace character after JSON data at line 2 column 1 of the JSON data
```
3. Firefox was also breaking when clearing the console, but with a different error :exploding_head:
```text
SyntaxError: JSON.parse: unterminated string at line 1 column 445 of the JSON data
```
4. At least Edge was behaving more like Chrome, but then I accidentally went fullscreen and then when I went back to normal, I got the same error as Chrome - later on also in Firefox nad Chrome
```text
SyntaxError: Unexpected token { in JSON at position 119
    at streamData (<anonymous>:20:40)
```
5. I could not reproduce the issue when using the Go client
6. In Safari I only got the issue after the 25 min period

So now I was confused and lost. But at no point did it occur to me that maybe, just maybe, the issue could be on the client. Even after [@lammel](https://github.com/lammel) had suggested that it's most likely the client (since it's the most complex part of my env). At least I did report the odd behaviour I was getting with fullscreen at [chromium/1305928](https://bugs.chromium.org/p/chromium/issues/detail?id=1305928).

And to add to the pain, [golang/go#51646](https://github.com/golang/go/issues/51646) got closed because my issue didn't seem like a bug (and it wasn't). So I got pretty frustrated with [@mengzhuo](https://github.com/mengzhuo) and I might have overreacted a bit in one of my replies :rolling_eyes: .

Around this time I was considering to just add a simple hack:
```js
...
try {
    const data = await res.json();
    if (typeof cb === 'function') {
    cb(data);
    }
} catch (e) {
    // check if e is a JSON parse err and continue to the next tick instead of returning
    return;
}
...
```

But then, about a day later, [@seankhliao](https://github.com/seankhliao) kindly responded with:
```text
Adding some extra logging, Go sends the output within a single write.
On the browser/js side, it's a data stream, and a single json message can be spread over 2 reads, eg I got:

{"pong":true,"reason":"the bug does not seem to be here :( where is it then? who knows? at this point, i have no clue! i'm lost 

and

now. i have no clue what to do. i need help. i don't know what i'm doing. where should i look for help? where could the bug be? is there even a bug? right now, i'm questioning everything!","date":"2022-03-14T15:20:09Z","quote":"dumbass! (that 70's show)"}

spread over 2 reads
```

The key information was `and a single json message can be spread over 2 reads`!

What I failed to understand while I was reading the [ReadableStreamDefaultReader](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultReader) documentation was that the stream chunks do not end on line boundaries (not necessarily). And combined with the [streaming response example](https://echo.labstack.com/cookbook/streaming-response/) from echo, I assumed that every message/data chunk I get from the reader is a JSON string (since that's what I was sending from the API).

Having that info, after a quick search, I found [handling text line by line](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultReader/read#example_2_-_handling_text_line_by_line) which made it easy to adjust the function I had to account for the JSON being split across multiple chunks. And it looks something like this:
```js
async function streamData(url) {
    const res = await fetch(url);

    if (!res.ok) {
        throw `got response w/ status code: ${res.status}`;
    }
    
    const reader = res.body.getReader();
    const t0 = window.performance.now();

    try {
        for await (const line of readLineByLine(reader)) {
          const json = await toJSON(line)
          console.info('Got data', json);
        }
        
        const t1 = window.performance.now();
        const duration = (t1 - t0)/1000;

        console.info(`Stream ended after ${duration}s`);
    } catch (e) {
        const t1 = window.performance.now();
        const duration = (t1 - t0)/1000;
        if (e.name === 'AbortError') {
          console.info(`Stream aborted after ${duration}s`);
        } else {
            console.info(`Stream errored after ${duration}s`);
            console.error(e);
        }
    }

    reader.cancel();
}

async function* readLineByLine(reader) {
    const utf8Decoder = new TextDecoder("utf-8");
    let {value: chunk, done: readerDone} = await reader.read();
    chunk = chunk ? utf8Decoder.decode(chunk, {stream: true}) : "";

    const re = /\r\n|\n|\r/gm;
    let startIndex = 0;

    for (; ;) {
        const result = re.exec(chunk);
        if (!result) {
            if (readerDone) {
                break;
            }

            const remainder = chunk.substr(startIndex);

            ({value: chunk, done: readerDone} = await reader.read());
            chunk = remainder + (chunk ? utf8Decoder.decode(chunk, {stream: true}) : "");
            startIndex = re.lastIndex = 0;

            continue;
        }

        yield chunk.substring(startIndex, result.index);

        startIndex = re.lastIndex;
    }

    if (startIndex < chunk.length) {
        // last line didn't end in a newline char
        yield chunk.substr(startIndex);
    }
}

async function toJSON(value) {
    try {
        const res = new Response(value);
        const data = await res.json();
        return data;
    } catch (e) {
        throw e;
    }
}
```

And that is how I wasted about 4 days of my life trying to fix a bug that wasn't there. Maybe it was ignorance, maybe something else. But what I have learned is that making assumptions about anything is not a good idea. If in doubt, spend more time reading about the topic in question or reach out to someone who may know more.

To see more details about the investigation, checkout [transcelestial/echo-streaming-bug](https://github.com/transcelestial/echo-streaming-bug).
