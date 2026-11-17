---
title: "The Streams API in JavaScript and Go"
description: "Checkout how to use the Streams API in JavaScript and how the API implementation would look like for it in Go."
date: 2020-11-08T20:24:00+08:00
lastmod: 2026-11-15T19:46:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["go", "javascript", "echo", "fetch", "streaming-api"]
categories: ["Go", "JavaScript"]
series: []

toc:
  enable: false
---

{{< admonition type=tip title="JSON Parsing" >}}
Checkout [Adventures with the Streaming API]({{< ref "/posts/adventures-with-the-streaming-api" >}}) to find out more about parsing JSON data from streaming APIs.
{{< /admonition >}}

If you ever had to build a realtime web app and you've built yourself a [REST](https://restfulapi.net/) backend (or you need to use same legacy REST backend), you most likely stumbled upon a pretty common issue: how do I stream a bunch of data from the backend to make it seem it's updated in realtime?

There's a bunch of ways to do this:

1. You can use [long polling](https://javascript.info/long-polling); but this may add some networking overhead (creating and tearing down connections every time you need data is not very efficient)
2. You can switch to [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API); but this means you need to reimplement your API
3. You can use the [Streams API](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API); you only need to update your API implementation to support it

The Streams API is currently a [LS](https://streams.spec.whatwg.org/) (living standard) and enjoys broad [browser support](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API#Browser_compatibility) across all modern browsers and runtimes.

For the purpose of illustrating how the server and client implementations will look like, we'll create a simple counter app that gets a new value every second from the backend.

## Server

We'll use [echo](https://echo.labstack.com/) to expose our counter endpoint:

```go
// main.go
package main

import (
  "encoding/json"
  "net/http"
  "time"

  "github.com/labstack/echo/v4"
  "github.com/labstack/echo/v4/middleware"
)

type Counter struct {
  Count int `json:"count"`
}

func main() {
  e := echo.New()
  e.Use(middleware.CORS())
  e.GET("/counter", getCounter)
  e.Logger.Fatal(e.Start(":9000"))
}

func getCounter(c echo.Context) error {
  ctx := c.Request().Context()

  c.Response().Header().Set(echo.HeaderContentType, echo.MIMEApplicationJSON)
  c.Response().WriteHeader(http.StatusOK)

  enc := json.NewEncoder(c.Response())

  var i = 0
  for {
    select {
    case <-ctx.Done():
      return nil
    default:
      counter := Counter{i}
      if err := enc.Encode(counter); err != nil {
        if ctx.Err() != nil {
          return nil
        }
        return err
      }
      c.Response().Flush()
      i++

      select {
      case <-ctx.Done():
        return nil
      case <-time.After(time.Second):
      }
    }
  }
}
```

To start it just run:

```bash
go run ./main.go
```

And if you run:

```bash
curl http://localhost:9000/counter
```

You should get a bunch of JSON messages:

```text
{"count":0}
{"count":1}
{"count":2}
{"count":3}
{"count":4}
{"count":5}
{"count":6}
...
```

## Client

We'll use [Vite](https://vitejs.dev/) (or `create-react-app`) to setup a simple app:

```bash
npm create vite@latest streams-api -- --template react
```

Now let's make a simple react hook that will just fetch and stream data from a URL (so we can reuse it with any endpoint):

```js
// src/stream.js
import {
  useCallback,
  useEffect,
  useRef,
  useState
} from 'react';

/**
 * Stream react hook
 * @param {object} params
 * @param {string} params.url
 */
export function useStream(params) {
  const [data, setData] = useState(null);
  const streamRef = useRef();
  const url = useRef();

  const stop = useCallback(() => {
    if (streamRef.current) {
      streamRef.current.abort();
    }
  }, []);

  useEffect(() => {
    if (url.current !== params.url) {
      url.current = params.url;
      if (streamRef.current) {
        streamRef.current.abort();
      }
      streamRef.current = new AbortController();
      startStream(params.url, data => setData(data), streamRef.current.signal);
    }

    return () => {
      if (streamRef.current) {
        streamRef.current.abort();
      }
    };
  }, [params.url]);

  return {data, stop};
}

/**
 * Use this function to start streaming data from an URL
 * @param {string} url
 * @param {Function} cb
 * @param {AbortSignal} sig
 */
export async function startStream(url, cb, signal) {
  const res = await fetch(url, {
    signal,
    method: 'GET'
  });

  const reader = res.body.getReader();

  while (true) {
    const {done, value} = await reader.read();
    if (done) {
      return;
    }

    const res = new Response(value);
    try {
      const data = await res.json();
      if (typeof cb === 'function') {
        cb(data);
      }
    } catch (e) {
      return;
    }
  }
}
```

{{< admonition type=warning title="JSON Parsing" >}}
While the above example may work most of the time when parsing the data as JSON, it's not guaranteed. And that's because the data may be split into separate chunks on the client side. So you'll need to account for that (see the tip at the top of this post for more info).
{{< /admonition >}}

{{< admonition type=tip title="NPM Package" >}}
You can find this hook on NPM too (checkout [rolandjitsu/react-fetch-streams](https://github.com/rolandjitsu/react-fetch-streams)).
{{< /admonition >}}

Now let's use this hook with our counter endpoint:

```js
// src/App.js
import {useMemo} from 'react';
import {useStream} from './stream';

function App(props) {
  const {data} = useStream({url: 'http://localhost:9000/counter'});
  const count = useMemo(() => data !== null ? data.count : 0, [data]);

  return (
    <div>
      <p>{count}</p>
    </div>
  );
}
```

And now run the app:

```bash
npm run dev
```

It won't look pretty, but you should be seeing a counter going up incrementally in the UI, similar to the curl call.

And that's about it. I hope this simple example made it a little more clear how to use the streams API. Thanks for reading.

For more code examples checkout [rolandjitsu/streams-api](https://github.com/rolandjitsu/streams-api).
