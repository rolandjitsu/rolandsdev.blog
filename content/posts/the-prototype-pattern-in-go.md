---
title: "The prototype pattern in Go"
description: "Implement the prototype design pattern in Golang."
date: 2020-10-25T18:21:00+08:00
lastmod: 2022-03-20T20:38:34+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---

This is a continuation of the *common design patterns I found in my old code* series, which I started in a previous post.

While going through some of the code I found a couple of instances where I was making copies of some structs, but I wasn't using the built in `copy()` method. I was, instead, using some custom copy logic.

The reason for that was that the struct had some properties that were slices of other structs and if I were to use `copy()`, it would get me in trouble as there was a possibility that the source struct could be mutated.

And this is actually what the [prototype](https://en.wikipedia.org/wiki/Prototype_pattern) design pattern is about. It's a design pattern that simply involves making copies of objects instead of creating new instances.

This sort of pattern can be useful when:
* creating instances of an object is time consuming or resource intensive (slow database queries, etc)
* making copies of an object is complex (deep copies, private members, etc - the same scenario as I described above)
* the object is exposed as an interface

To get an idea of what it looks like in practice, consider the following example:
```go
obj := NewObj(1)
copy := obj.Clone()

func NewObj(prop int) MyObj {
    return &myObj{PubProp: prop}
}

type MyObj interface {
    Clone() MyObj
}

type myObj struct {
    PubProp int
    privProp int
}

func (o *myObj) Clone() *MyObj {
    return &myObj{
        PubProp: o.Prop,
        privProp: o.privProp,
    }
}
```

The above example has 2 main reasons for applying the prototype design pattern:
1. Our object is an interface
2. There's a private property

That's more or less all there is to this pattern.

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
