---
title: "The singleton pattern in Go"
description: "Implement the singleton design pattern in Golang."
date: 2020-10-24T20:17:00+08:00
lastmod: 2022-03-20T20:24:02+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---

I recently started going through some of my old code and I was trying to identify some common design patterns. I thought it could be a good memory exercise and refresher on [software design patterns](https://www.geeksforgeeks.org/software-design-patterns/) as it's been quite some time since I last read through that.

And while I was doing that, I thought it might be a good idea to write about the patterns with the most occurrences.

One of the patterns I seem to be using the most is the [singleton](https://www.geeksforgeeks.org/singleton-design-pattern/) pattern. It's a design pattern that restricts the instantiation of a class (struct in Go) to a single object.

What that means is that you will share the same struct instance throughout your program. And the simplest example of this are database connections.

Say you're building an API that needs to query some data from somewhere. And the database you use doesn't allow for multiple connections from the same host.

A simple approach to this is to share the connection, so you'd end up with something similar to the following:
```go
// cmd/api/main.go
package main

import (
  "github.com/labstack/echo/v4"
  
  dataApi "github.com/my-org/my-project/cmd/api/data"
  "github.com/my-org/my-project/pkg/db"
)

func main() {
  c := &db.Conn{"my-db.org:5000"}
  e := echo.New()
  
  e.GET("/data", dataApi.MakeGet(c))
  
  ...
}
```

```go
// pkg/db/conn.go
package db

type Conn struct {
  Address string
}

func (c *Conn) Query(q string) ([]byte, error) {
  ...
}
```

```go
// cmd/api/data/get.go
package data

import (
  "github.com/labstack/echo/v4"
  
  "github.com/my-org/my-project/pkg/db"
)

func MakeGet(c *db.Conn) func (echo.Context) error {
  return func (ctx echo.Context) error {
    data, err := c.Query("SELECT * FROM data;")
    ...
  }
}
```

That's pretty much it. Of course, there's probably better ways to manage this, e.g create a context and share the connection through that, but the above is a very simple illustration of what this design pattern looks like in practice.

I hope you found this useful.

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
