---
title: "The iterator pattern in Go"
description: "Implement the iterator design pattern in Golang."
date: 2020-10-25T18:59:00+08:00
lastmod: 2022-03-20T20:44:44+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---

This is a continuation of the *common design patterns I found in my old code* series, which I started in a previous post.

The [iterator](https://en.wikipedia.org/wiki/Iterator_pattern) pattern is a frequently used design pattern in software and it's very simple. It entails that a collection must provide an iterator that can be used to iterate through its objects.

To put it in simple terms:
```go
c := MyCollection{}

for c.Next() {
    v := c.Value()
    ...
}
```

Though, I haven't seen this used very often in Go (it doesn't mean it's true). I could only find a single instance of this while going through my code (a Firestore [DocumentIterator](https://pkg.go.dev/cloud.google.com/go/firestore#DocumentIterator)).

Most of the time, I use [channels](https://tour.golang.org/concurrency/2) if performance is not a concern:
```go
docs := make(chan Document)

go func(c chan<- Document) {
  // fetch documents async from somewhere
  ...
  c <- doc
}(docs)

for doc := range docs {
  ...
}
```

But if performance is important, channels are not a good idea, and you'd probably be better off if you implement your own iterator (this is debatable - so don't take my word for it).

Let's say you want a simple way to create a range of numbers between two intervals and iterate over that. You could write something like:
```go
myRange := &IntRange{Start: 100, End: 200}

for {
  if i, done = r.Next(); done {
    break
  }
  // do something with the int 
 } 

type IntRange struct {
  Start int
  End   int
}

func (r *IntRange) Next() (v int, done bool) {
  r.Start++
  v = r.Start
  if r.Start > r.End {
    done = true
    return
  }
  return
}
```

And while the above is a silly example (because you could just use a for loop and start from `Start` to `End`), it illustrates how you could implement an iterator in Go.

Thanks for reading and I hope you found this useful.

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
