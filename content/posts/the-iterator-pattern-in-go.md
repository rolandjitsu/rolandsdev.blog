---
title: "The iterator pattern in Go"
description: "Implement the iterator design pattern in Golang, from custom structs to Go 1.23 range-over-func iterators."
date: 2020-10-25T18:59:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: true
---

The [iterator](https://en.wikipedia.org/wiki/Iterator_pattern) pattern decouples collection traversal from the underlying representation. In classical OOP, a collection exposes an iterator object with `Next()` and `Value()` methods:

```go
c := MyCollection{}

for c.Next() {
    v := c.Value()
    // process v
}
```

Traditionally in Go, you had three options for iteration:
1. Index-based loops over slices or arrays (`for i, v := range slice`).
2. Iterating over channels (`for doc := range docs`), which works well for asynchronous streaming but adds synchronization overhead and goroutine allocations.
3. Custom iterator structs with a `Next()` method (such as Google Cloud's `firestore.DocumentIterator` or `database/sql.Rows`).

## Custom Struct Iterators

A custom stateful iterator struct looks like this:

```go
type IntRange struct {
  current int
  end     int
}

func NewIntRange(start, end int) *IntRange {
  return &IntRange{current: start, end: end}
}

func (r *IntRange) Next() (int, bool) {
  if r.current > r.end {
    return 0, false
  }
  val := r.current
  r.current++
  return val, true
}
```

Consuming it requires manual loop control:

```go
r := NewIntRange(100, 200)
for {
  val, ok := r.Next()
  if !ok {
    break
  }
  fmt.Println(val)
}
```

While functional, it cannot integrate with Go's native `for ... := range` syntax.

## Modern Go: Range-Over-Func Iterators (Go 1.23+)

Go 1.23 added native support for **range-over-function iterators** and the standard [`iter`](https://go.dev/pkg/iter/) package. Instead of stateful structs or channel coordination, an iterator is simply a function that accepts a `yield` callback.

### Single-Value Iterators (`iter.Seq[V]`)

An iterator yielding single values implements the signature `func(yield func(V) bool)`:

```go
package main

import (
  "fmt"
  "iter"
)

func IntRange(start, end int) iter.Seq[int] {
  return func(yield func(int) bool) {
    for i := start; i <= end; i++ {
      if !yield(i) {
        return // consumer stopped early (break)
      }
    }
  }
}

func main() {
  // Consumed directly with native for-range!
  for n := range IntRange(100, 105) {
    fmt.Println(n)
  }
}
```

When the loop encounters a `break` statement, `yield` returns `false`, allowing the producer function to exit cleanly and release any open resources (like file handles or network connections).

### Key-Value Iterators (`iter.Seq2[K, V]`)

When you need index/value or key/value pairs, use `iter.Seq2[K, V]`:

```go
func FilteredIndexed[T any](items []T, predicate func(T) bool) iter.Seq2[int, T] {
  return func(yield func(int, T) bool) {
    idx := 0
    for _, item := range items {
      if predicate(item) {
        if !yield(idx, item) {
          return
        }
        idx++
      }
    }
  }
}
```

### Standard Library Integration

Range functions compose seamlessly with standard library utilities like [`slices.Collect`](https://go.dev/pkg/slices/#Collect):

```go
// Materialize iterator output directly into a slice
evens := slices.Collect(func(yield func(int) bool) {
  for n := range IntRange(1, 10) {
    if n%2 == 0 && !yield(n) {
      return
    }
  }
})
fmt.Println(evens) // [2 4 6 8 10]
```

Range-over-func combines the zero-allocation performance of custom loops with the elegance of native `for-range` traversal, making custom iterator structs largely obsolete for new Go code.

For more design pattern examples, checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
