---
title: "The observer pattern in Go"
description: "Implement the observer design pattern in Golang."
date: 2020-10-31T12:56:00+08:00
lastmod: 2022-03-20T21:01:17+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---
This is a continuation of the *common design patterns I found in my old code* series, which I started in a previous post.

The [observer](https://en.wikipedia.org/wiki/Observer_pattern) pattern is a design pattern in which an object (a **subject**) keeps track of all of its dependents (**observers**) and notifies them of any state changes.

In Go, the closest example of this pattern are the builtin [channels](https://tour.golang.org/concurrency/2) and the use of [goroutines](https://gobyexample.com/goroutines):
```go
sub := make(chan interface{})

go func(c <-chan interface{}>) {
    for data := range c {
        fmt.Println(data)
    }
}(sub)

sub <- "Hey there"
```

Though, for multiple observers to be notified, you need to send the message once for each observer.

Another example is the [rxjs](https://www.learnrxjs.io) lib and accompanying libs for other languages.

And to illustrate how an implementation of this pattern looks like, check the following example:
```go
sub := NewSubject()

sub1 := sub.Subscribe(func (data interface{}) {
    fmt.Println("Sub 1 says", data)
})
sub2 := sub.Subscribe(func (data interface{}) {
    fmt.Println("Sub 2 says", data)
})

sub.Next("Hey there")

sub2.Unsubscribe()

sub.Next("Hey again")

func NewSubject() Subject {
    // ...
}

type Subject interface {
    Subscribe(func(interface{})) Subscription
    Next(interface{})
}

type Subscription interface {
    Unsubscribe()
}
```

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
