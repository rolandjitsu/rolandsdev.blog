---
title: "The adapter pattern in Go"
description: "Implement the adapter design pattern in Golang."
date: 2020-10-26T18:36:00+08:00
lastmod: 2022-03-20T20:56:14+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---

This is a continuation of the *common design patterns I found in my old code* series, which I started in a previous post.

A while back I was playing around with a [Raspberry Pi 4](https://www.raspberrypi.org/) and some temperature sensors. Some of the sensors I was using were exposing data over the [SPI](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface) communication interface and some were exposing it over the [I2C](https://en.wikipedia.org/wiki/I%C2%B2C) interface.

But I didn't want the client app that was reading the temperature to change every time I swapped a sensor, so I ended up making a simple wrapper that exposed a single method to read the temperature and which could be initialized with different protocols.

This is the [adapter](https://en.wikipedia.org/wiki/Adapter_pattern) design pattern. It's a design pattern that allows the interface of a class or object to be used as another interface.

This is pretty common in hardware drivers. Take printers for instance. Most printers can be used either via USB or serial connections or over the network.

It's also a frequently used pattern in ORM libraries ([ActiveRecord](https://guides.rubyonrails.org/active_record_basics.html), [GORM](https://gorm.io/index.html), etc), because it allows for connections and queries to different data backends (databases) while keeping the same client interface.

```go
bkd := &SQLBackend{}
db := &Conn{Backend: bkd}

db.Query()

...

type Conn struct {
  Backend Backend
}

func (c *Conn) Query() ([]byte, error) {
  return c.Backend.Query()
}

type Backend interface {
  Query() ([]byte, error)
}

type SQLBackend struct {}

func (db *SQLBackend) Query() ([]byte, error) {
  ...
}

type NoSQLBackend struct {}

func (db *NoSQLBackend) Query() ([]byte, error) {
  ...
}
```

And that's all there is to it.

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
