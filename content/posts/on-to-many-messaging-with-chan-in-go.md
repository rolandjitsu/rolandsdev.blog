---
title: "One-to-many messaging with chan in Go"
description: "A simple implementation of one-to-many (or many-to-many) communication using channels in Go."
date: 2022-04-02T08:07:18+08:00
lastmod: 2022-04-02T08:07:18+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "pub-sub"]
categories: ["Go"]
series: []

toc:
  enable: no
---

The [chan](https://go.dev/ref/spec#Channel_types) type in Go is a mechanism which can be used to send or receive data from one function to another. E.g:
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

Channels can be *bidirectional* (`chan string`) or *directional* (`chan <- string` sender/`<-chan string` receiver). And they operate like FIFO queues.

There's also *unbuffered* (`make(chan string)`) and *buffered* (`make(chan string, 10)`) channels. The difference b/t the 2 is that buffered channels will not block on send until the buffer is full. E.g:
```go
func main() {
    c := make(chan string, 2)
    // this won't block
    c <- "hi"
    c <- "there"
    
    // this will block because the buffer size is 2
    // if we'd change the size/capacity to 3, this send
    // won't block either
    c <- ":)"
}
```

Similarly, the receiver end won't block until the buffer is empty:
```go
func main() {
    c := make(chan string, 2)
    c <- "hi"
    c <- "there"
    fmt.Printf("%s %s", <-c, <-c)
}
```

Another important feature of channels is that if there are many receivers, only one of the receivers will get the message (so a one-to-one or many-to-one data flow). E.g:
```go
func main() {
    var wg sync.WaitGroup
    c := make(chan string)
    wg.Add(1)
    go printHello(c, 1, &wg)
    wg.Add(1)
    go printHello(c, 2, &wg)
    sayHello(c)
    wg.Wait() // this will deadlock
}

func sayHello(msgChan chan<- string) {
    msgChan <- "hello"
}

func printHello(msgChan <-chan string, id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Println(fmt.Sprintf("%s from %d", <-msgChan, id))
}
```

This can be a useful feature, but sometimes you may need a one-to-many/many-to-many communication channel, and unfortunately, this is not available natively in Go as of [1.18](https://go.dev/blog/go1.18).

So you will most likely end up using a pkg like [RxGo](https://github.com/ReactiveX/RxGo) or implement your own one-to-many/many-to-many interface.

I usually tend to use whatever pkgs I find that provide the features I'm interested in, but sometimes the implementation is complicated and difficult to debug. Or sometimes I just want to learn how it's done, and I end up implementing functionality from scratch (not the wisest thing to do :laughing: ).

So how would you go about implementing the aforementioned pattern? Well, it's not that complicated. First, let's start with an interface:
```go
// NewPiper creates a piper interface that will
// pipe data from a source chan to many receivers.
// 
// you could argue returning a struct,
// but I find using interfaces much more flexible,
// because we can create mocks for them using https://github.com/golang/mock.
func NewPiper(source <-chan Data) Piper {
    return nil
}

type Piper interface {
	Pipe() <-chan Data 
}

// we're not using generics yet,
// but we want to abstract the data type for reusability
type Data struct {
    V interface{}
}
```

So that looks pretty simple. But how about the implementation? That's not complicated either.
There's a few ways to go about it, and the one I find most easy to understand is using some sort of bookkeeping. What I mean by that is that we will need to return a new channel every time `Pipe()` is called and then keep track of these channels so that we can send messages to each when the source channel sends.

So let's create a struct that implements our interface:
```go
func NewPiper(source <-chan Data) Piper {
    return &piper{
        chans: []chan<- Data{},
    }
}

type piper struct {
	chans []chan<- Data
}

func (p *piper) Pipe() <-chan Data {
    c := make(chan Data)
	p.chans = append(p.chans, c)
	return c
}
```

Then let's add some logic to read from the source and forward the messages to our channels:
```go
func NewPiper(source <-chan Data) Piper {
	p := &piper{
		chans: []chan<- Data{},
	}
	go p.setupRcvLoop(source)
	return p
}

func (p *piper) setupRcvLoop(source <-chan Data) {
	for d := range source {
		for _, c := range p.chans {
			c <- d
		}
	}
}
```

We should probably also make sure we close the channels we create when the source is closed:
```go
func (p *piper) setupRcvLoop(source <-chan Data) {
	for d := range source {
		for _, c := range p.chans {
			c <- d
		}
	}
	// source got closed, so close the receivers
	for _, c := range p.chans {
		close(c)
	}
}
```

And maybe address concurrency concerns:
```go
type piper struct {
	mux   sync.RWMutex
	chans []chan<- Data
}

func (p *piper) Pipe() <-chan Data {
	p.mux.Lock()
	defer p.mux.Unlock()
	c := make(chan Data)
	p.chans = append(p.chans, c)
	return c
}

func (p *piper) setupRcvLoop(source <-chan Data) {
	for d := range source {
		p.mux.RLock()
		for _, c := range p.chans {
			c <- d
		}
		p.mux.RUnlock()
	}
	// source got closed, so close the receivers
	for _, c := range p.chans {
		close(c)
	}
}
```

So far, so good. Let's test it:
```go
func main() {
    // the source
    c := make(chan Data)
    
    // the piping
    piper := NewPiper(c)
    p1 := piper.Pipe()
    p2 := piper.Pipe()
    
    // need to wait for msgs before we exit
    var wg sync.WaitGroup

    wg.Add(1)
    go func(){
        defer wg.Done()
        fmt.Println(<-p1)
    }()

    wg.Add(1)
    go func(){
        defer wg.Done()
        fmt.Println(<-p2)
    }()

    c <- Data{"hey"}
    wg.Wait()
}
```

We should be seeing something like (check https://go.dev/play/p/DG8eJacsSGl):
```text
{hey}
{hey}
```

Great. It seems to work. But what if one of the receivers blocks because it's doing some work? Let's see what happens:
```go
func main() {
    // the source
    c := make(chan Data)
    
    // the piping
    piper := NewPiper(c)
    p1 := piper.Pipe()
    p2 := piper.Pipe()
    
    // need to wait for msgs before we exit
    var wg sync.WaitGroup

    wg.Add(1)
    go func(){
        defer wg.Done()
        time.Sleep(time.Second)
        fmt.Println(<-p1, time.Now())
    }()

    wg.Add(1)
    go func(){
        defer wg.Done()
        fmt.Println(<-p2, time.Now())
    }()

    c <- Data{"hey"}
    wg.Wait()
}
```

It still works, but it doesn't look right (check https://go.dev/play/p/gfjNV3dHfvU):
```text
{hey} 2009-11-10 23:00:01 +0000 UTC m=+1.000000001
{hey} 2009-11-10 23:00:01 +0000 UTC m=+1.000000001
```

It seems like both pipes received the msg at the same time. That's not exactly great :disappointed: . But that kind of makes sense. If we take a closer look at:
```go
func (p *piper) setupRcvLoop(source <-chan Data) {
	for d := range source {
		p.mux.RLock()
		for _, c := range p.chans {
			c <- d
		}
		p.mux.RUnlock()
	}
	// source got closed, so close the receivers
	for _, c := range p.chans {
		close(c)
	}
}
```

We're getting blocked by the first pipe because the send action blocks until there's a receiver. So how do we fix this? Well, the simplest way to address this is to discard messages if there are no receivers - not ideal for every scenario, but it'll do (see https://go.dev/play/p/r3s8lJ_JvX1):
```go
func (p *piper) setupRcvLoop(source <-chan Data) {
	for d := range source {
		p.mux.RLock()
		for _, c := range p.chans {
			select {
            case c <- d:
            default:
            }
		}
		p.mux.RUnlock()
	}
	// source got closed, so close the receivers
	for _, c := range p.chans {
		close(c)
	}
}
```

Ok. That didn't work either :exploding_head: ! We're getting a deadlock. And this makes sense as well, because we add our receiver too late, and we missed the message. This is not easy to fix. I mean, there's an easy fix, but it may not be ideal either.

We just need to use a buffered channel with size 1 when we create the receivers/pipes (see https://go.dev/play/p/Ruqmk3QKQT1):
```go
func (p *piper) Pipe() <-chan Data {
	p.mux.Lock()
	defer p.mux.Unlock()
	c := make(chan Data, 1)
	p.chans = append(p.chans, c)
	return c
}
```

And now we get what we expect:
```text
{hey} 2009-11-10 23:00:00 +0000 UTC m=+0.000000001
{hey} 2009-11-10 23:00:01 +0000 UTC m=+1.000000001
```

And that's more or less how you can build a simple one-to-many/many-to-many data flow using `chan` in Go. It's obviously a lot more complex as you start thinking about different use cases or scenarios, but hopefully the above is good enough as a guide/baseline.

Do checkout the [transcelestial/chanpiper](https://github.com/transcelestial/chanpiper) pkg if the above is good enough for your use case.
