---
title: "The factory method pattern in Go"
description: "Implement the factory method design pattern in Golang."
date: 2020-10-25T17:50:00+08:00
lastmod: 2022-03-20T20:31:25+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---

This is a continuation of the *common design patterns I found in my old code* series, which I started in a previous post.

Another common pattern I found is the [factory method pattern](https://en.wikipedia.org/wiki/Factory_method_pattern), which is a design pattern used to create different types of objects using the same interface.

This pattern is actually pretty common in Go. Some good example of this are the builtin I/O libraries:
```go
package main

import (
  "bytes"
  "encoding/csv"
  "fmt"
  "io"
  "os"
)

func main() {
  f, _ := os.Open("avengers.csv")
  records, _ := parseCsv(f)
  fmt.Println("Records from file", records)
  
  data := []byte("name,surname\nJohn,Snow\n")
  r := bytes.NewReader(data)
  records, _ = parseCsv(r)
  fmt.Println("Records from bytes", records)
}

func parseCsv(r io.Reader) ([][]string, error) {
  cr := csv.NewReader(r)
  return cr.ReadAll()
}
```

Both the `bytes.NewReader()` and `os.Open()` implement the `io.Reader` interface which comes in handy for the `parseCsv()` method above as we can use it to parse data from multiple sources.

Another scenario when using this design pattern is useful are in unit tests:
```go
// somewhere in your code
func UseMyObj(o MyObj) []byte {
  return o.Read()
}

type MyObj interface {
  Read() []byte
}

// the unit test for the above fn
func TestUseMyObj(t *testing.T) {
  o := NewMockObj([]byte("hello"))
  b := UseMyObj(o)
  ...
}

func NewMockObj(data []byte) MyObj {
  return &mockMyObj{data}
}

type mockObj struct {
  data []byte
}

func (o *mockObj) Read() []byte {
  return o.data
}
```

By making an interface for `MyObj`, we can mock the behaviour of the object in unit tests. You'll probably find this pattern a lot in open source libraries (e.g [periph.io](https://github.com/google/periph)) that provide some sort of testing utils.

Another common occurrence of this pattern are factory functions which are useful when you need to create objects with some defaults:
```go
makeObj := MakeObjFactory("A")
obj1 := makeObj(1)
obj2 := makeObj(2)

func MakeObjFactory(a string) func(b int) MyObj {
  return func() MyObj {
    return MyObj{
      PropA: a,
      PropB: b,
    }
  }
}

type MyObj struct {
  PropA string
  PropB int
}
```

The above are just a few examples of how the factory design pattern is used in Go, but I hope it's enough to get an idea of how to use it in practice.

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
