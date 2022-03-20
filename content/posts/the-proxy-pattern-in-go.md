---
title: "The proxy pattern in Go"
description: "Implement the proxy design pattern in Golang."
date: 2020-11-07T13:13:00+08:00
lastmod: 2022-03-20T21:05:02+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---

This is a continuation of the *common design patterns I found in my old code* series, which I started in a previous post.

The [proxy](https://en.wikipedia.org/wiki/Proxy_pattern) pattern is a design pattern in which a class (**proxy**) acts as an interface to something else. The proxy could be an interface to anything: a network connection, another class, a file, etc.

A proxy can be useful in a variety of situations:
* A frontend for load balancing
* Hide private infrastructure
* Caching layer
* etc

A good example of a proxy that can be used as a load balancer (and other purposes) is [nginx](https://www.nginx.com/) or [net/http/httputil](https://pkg.go.dev/net/http/httputil) (the `ReverseProxy`).

To illustrate how to use this pattern in Go, let's implement a simple caching layer for files to speed up multiple reads for the same files:
```go
package fileproxy

import (
  "io"
  "io/ioutil"
  "sync"
)

var mux sync.RWMutex
var cache map[string][]byte

func init() {
  cache = map[string][]byte{}
}

func NewFileProxy(name string) FileProxy {
  return &fileProxy{Name: name}
}

type FileProxy interface {
  io.Reader
}

type fileProxy struct {
  Name string
}

func (fp *fileProxy) Read(b []byte) (n int, err error) {
  mux.RLock()
  if data, ok := cache[fp.Name]; ok {
    copy(b, data)
    mux.RUnlock()
    n = len(b)
    return
  }
 
  mux.RUnlock()

  var data []byte
  data, err = ioutil.ReadFile(fp.Name)
  if err == nil {
    copy(b, data)
    mux.Lock()
    n = len(b)
    cache[fp.Name] = data
    mux.Unlock()
  }

  return
}

var _ FileProxy = &fileProxy{}
```

And if you'd run a simple benchmark:
```go
func BenchmarkReadFile(b *testing.B) {
  for n := 0; n < b.N; n++ {
    _, err := ioutil.ReadFile("somefile.txt")
    if err != nil {
      b.Fatal(err)
    }
  }
}

func BenchmarkFileProxy(b *testing.B) {
  data := make([]byte, 24)
  fp := NewFileProxy("somefile.txt")

  for n := 0; n < b.N; n++ {
    _, err := fp.Read(data)
    if err != nil {
      b.Fatal(err)
    }
  }
}
```

```text
BenchmarkReadFile-8        87010             13601 ns/op
BenchmarkFileProxy-8    52101362                21.9 ns/op
```

You can see how the proxy could improve your app performance in a similar situation.

And that's it. I hope this example has shed some light on how the proxy pattern works and can be used to improve your code performance.

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
