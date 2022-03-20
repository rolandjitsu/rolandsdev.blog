---
title: "The command pattern in Go"
description: "Implement the command design pattern in Golang."
date: 2022-03-20T20:51:18+08:00
lastmod: 2022-03-20T20:51:18+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["go", "design-patterns"]
categories: ["Go"]
series: ["Design Patterns in Go"]

toc:
  enable: no
---

This is a continuation of the *common design patterns I found in my old code* series, which I started in a previous post.

The [command](https://en.wikipedia.org/wiki/Command_pattern) pattern is a design pattern that encapsulates an action or request as an object that can be parameterized. And it's commonly associated with terms like receiver, command, invoker and client.

Usually, the invoker doesn't know anything about the implementation details of the command or receiver, it just knows the command interface and its only responsibility is to invoke the command and optionally do bookkeeping of commands.

The command is the object that knows about the receiver and its responsibility is to execute methods on the receiver.

Go's [exec](https://golang.org/pkg/os/exec/) package is an example of this pattern:
```go
package main

import (
  "os/exec"
)

func main() {
  cmd := exec.Command("sleep", "1")
  err := cmd.Run()
}
```

Or the [net/http](https://golang.org/pkg/net/http/) package:
```go
package main

import (
  "http/net"
)

func main() {
  c := &http.Client{}
  req, err := http.NewRequest("GET", "http://example.com", nil)
  res, err := c.Do(req)
}
```

Though, it's not straightforward to see which is the invoker and receiver.

But take the following example:
```go
p := &Pinger{}
ping := &Ping{}
pingCmd = &PingCmd{ping}

p.Execute(pingCmd)

type Pinger struct {}

func (p *Pinger) Execute(c Cmd) {
  c.Execute()
  ...
}

type Cmd interface {
  Execute()
}

type Ping struct {}

func (p *Ping) Send() {
  ...
}

type PingCmd struct {
  Ping *Ping
}

func (pc *PingCmd) Execute() {
  pc.Ping.Send()
}
```

It's easy to see that:
1. `Pinger` is the invoker
2. `Ping` is the receiver
3. `PingCmd` is the command

What this pattern essentially allows us is to decouple the action/request logic from the execution. This can be useful in a variety of situations:
* [Undo](https://en.wikipedia.org/wiki/Undo) interactions
* Remote execution
* Parallel processing
* Micro-service architectures
* Task queues
* etc

For more design pattern examples, please checkout [rolandjitsu/go-design-patterns](https://github.com/rolandjitsu/go-design-patterns).
