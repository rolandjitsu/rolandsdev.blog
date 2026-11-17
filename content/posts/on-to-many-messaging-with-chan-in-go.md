---
title: "One-to-many messaging with chan in Go"
description: "A simple implementation of one-to-many (or many-to-many) communication using channels and generics in Go."
date: 2022-04-02T08:07:18+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["go", "pub-sub", "concurrency"]
categories: ["Go"]
series: []

toc:
  enable: true
---

The [`chan`](https://go.dev/ref/spec#Channel_types) type in Go is a synchronization and communication mechanism used to send or receive data between concurrent goroutines:

```go
func main() {
    c := make(chan string)
    go printHello(c)
    sayHello(c)
}

func sayHello(msgChan chan<- string) {
    msgChan <- "hello"
}

func printHello(msgChan <-chan string) {
    fmt.Println(<-msgChan)
}
```

Channels can be *bidirectional* (`chan string`) or *directional* (`chan<- string` send-only, `<-chan string` receive-only), and they operate as FIFO queues.

There are also *unbuffered* (`make(chan string)`) and *buffered* (`make(chan string, 10)`) channels. Unbuffered sends block until a receiver is ready. Buffered channels will not block on send until the internal buffer capacity is full:

```go
func main() {
    c := make(chan string, 2)
    c <- "hi"
    c <- "there" // buffer is full

    // A third send would block until a receiver reads from c
}
```

## The Single-Receiver Constraint

A foundational design characteristic of Go channels is that **each sent message is delivered to exactly one receiver**. If multiple goroutines listen on the same channel, messages are distributed across them like a worker pool:

```go
func main() {
    var wg sync.WaitGroup
    c := make(chan string)

    wg.Add(2)
    go worker(1, c, &wg)
    go worker(2, c, &wg)

    c <- "task 1"
    c <- "task 2"
    close(c)
    wg.Wait()
}

func worker(id int, ch <-chan string, wg *sync.WaitGroup) {
    defer wg.Done()
    for msg := range ch {
        fmt.Printf("worker %d received: %s\n", id, msg)
    }
}
```

Each item sent to `c` is consumed by only one worker. But what if your architecture requires **broadcasting**: delivering every message from a single publisher to *all* active subscribers (a 1-to-many or pub-sub pattern)?

## Building a Generic Broadcaster (`Piper[T]`)

With Go generics, we can build a reusable, type-safe fan-out broadcaster without boxing values into `any` / `interface{}`.

### 1. Interface Definition

We define an interface that provides a subscription method and clean teardown:

```go
type Piper[T any] interface {
    // Pipe creates and returns a new subscription channel.
    Pipe() <-chan T

    // Unpipe removes a subscription channel and closes it.
    Unpipe(ch <-chan T)
}
```

### 2. Implementation with Goroutine Safety

We maintain a slice or map of subscriber channels and a read-write mutex to prevent race conditions during concurrent subscribe, publish, and teardown operations:

```go
type piper[T any] struct {
    mux   sync.RWMutex
    chans map[chan T]struct{}
}

func NewPiper[T any](source <-chan T) Piper[T] {
    p := &piper[T]{
        chans: make(map[chan T]struct{}),
    }
    go p.listen(source)
    return p
}

func (p *piper[T]) Pipe() <-chan T {
    p.mux.Lock()
    defer p.mux.Unlock()

    // Use a small buffer to avoid immediate blocking on bursty sends
    c := make(chan T, 1)
    p.chans[c] = struct{}{}
    return c
}

func (p *piper[T]) Unpipe(ch <-chan T) {
    p.mux.Lock()
    defer p.mux.Unlock()

    for c := range p.chans {
        if c == ch {
            delete(p.chans, c)
            close(c)
            break
        }
    }
}
```

### 3. The Broadcast Loop and Clean Shutdown

The broadcast loop consumes from the source channel. When broadcasting, we use a `select` with a `default` branch to prevent a single slow or stalled consumer from blocking all other subscribers (non-blocking drop):

```go
func (p *piper[T]) listen(source <-chan T) {
    for msg := range source {
        p.mux.RLock()
        for ch := range p.chans {
            select {
            case ch <- msg:
            default:
                // Buffer full; drop message or log backpressure
            }
        }
        p.mux.RUnlock()
    }

    // Source closed: close all active subscriber channels under write lock
    p.mux.Lock()
    for ch := range p.chans {
        close(ch)
    }
    p.chans = nil
    p.mux.Unlock()
}
```

## Example: Broadcasting to Multiple Subscribers

```go
func main() {
    source := make(chan string)
    broadcaster := NewPiper(source)

    sub1 := broadcaster.Pipe()
    sub2 := broadcaster.Pipe()

    var wg sync.WaitGroup
    wg.Add(2)

    go func() {
        defer wg.Done()
        for msg := range sub1 {
            fmt.Println("sub1 received:", msg)
        }
    }()

    go func() {
        defer wg.Done()
        for msg := range sub2 {
            fmt.Println("sub2 received:", msg)
        }
    }()

    source <- "event: user_login"
    source <- "event: metrics_flush"
    close(source)

    wg.Wait()
}
```

Both `sub1` and `sub2` receive copies of each published event:

```text
sub1 received: event: user_login
sub2 received: event: user_login
sub1 received: event: metrics_flush
sub2 received: event: metrics_flush
```

## Design Trade-offs: Blocking vs Dropping

When distributing messages to multiple channels, consider your application's tolerance for slow consumers:

1. **Blocking broadcast (`ch <- msg`)**: Ensures zero message loss, but the slowest subscriber throttles the entire pipeline.
2. **Non-blocking drop (`select { case ch <- msg: default: }`)**: Protects system throughput and responsive consumers, but slow consumers lose messages when their buffers fill.
3. **Dynamic ring buffer**: Uses an intermediate queue per subscriber, disconnecting or logging lagging clients before exceeding memory boundaries.

For a battle-tested library implementation, checkout [transcelestial/chanpiper](https://github.com/transcelestial/chanpiper).
