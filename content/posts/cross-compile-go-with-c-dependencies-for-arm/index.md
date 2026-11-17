---
title: "Cross-compile Go with C dependencies for ARM"
description: "Cross-compile Go packages that call C code with cgo for ARM."
date: 2020-10-11T17:32:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["go", "docker", "cross-compile", "arm"]
categories: ["DevOps", "Go"]
series: []

toc:
  enable: true
---

Go includes first-class support for cross-compiling pure Go code to multiple operating systems and architectures simply by specifying the `$GOOS` and `$GOARCH` environment variables (see the official [environment list](https://go.dev/doc/install/source#environment)).

However, when your project uses C libraries or legacy system APIs via [cgo](https://go.dev/cmd/cgo/), compilation requires a matching C cross-compiler and target C runtime headers. Configuring this directly on your host machine can quickly lead to conflicting toolchains and package versions.

Using Docker multi-stage builds and Buildx provides a reproducible, hermetic cross-compilation pipeline for both 64-bit ARM (`arm64` / `aarch64`) and 32-bit ARM (`armhf`).

## Prerequisites

1. [Docker](https://docs.docker.com/engine) >= `20.10.0`
2. [buildx](https://github.com/docker/buildx) >= `v0.8.0`

Bootstrap a container builder if you haven't already:

```bash
docker buildx create --name arm-builder --driver docker-container --use
docker buildx inspect --bootstrap
```

## Setup the Go Project

Initialize a module:

```bash
mkdir go-cross && cd go-cross
go mod init github.com/my-username/go-cross
```

Create a C-backed Go package:

```go
// pkg/hello/hello.go
package hello

// #include <stdlib.h>
// #include "hello.h"
import "C"
import (
	"fmt"
	"unsafe"
)

func Greet(name string) {
	cs := C.CString(name)
	size := C.size_t(len(name))
	msg := C.greet(cs, size)
	fmt.Println(C.GoString(msg))
	C.free(unsafe.Pointer(cs))
}
```

The accompanying C header and source:

```c
// pkg/hello/hello.h
#ifndef HELLO_H_
#define HELLO_H_

#include <stdio.h>
#include <stdlib.h>

char *greet(const char *name, size_t size);

#endif
```

```c
// pkg/hello/hello.c
#include "hello.h"

char *greet(const char *name, size_t size) {
  char *msg = malloc(size + 16);
  if (!msg) return NULL;
  sprintf(msg, "Hello, %s!", name);
  return msg;
}
```

And the application entrypoint:

```go
// cmd/hello/main.go
package main

import (
	"flag"

	"github.com/my-username/go-cross/pkg/hello"
)

func main() {
	flag.Parse()

	name := flag.Arg(0)
	if name == "" {
		name = "stranger"
	}

	hello.Greet(name)
}
```

## Multi-Architecture Dockerfile

The Dockerfile installs target cross-toolchains from Debian Bookworm and uses BuildKit cache mounts for fast incremental rebuilds:

```dockerfile
# syntax=docker/dockerfile:1
# Dockerfile.hello
ARG GO_VERSION=1.23
FROM golang:${GO_VERSION}-bookworm AS toolchain

ARG TARGETARCH=arm64

RUN apt-get update && \
    if [ "$TARGETARCH" = "arm64" ]; then \
        apt-get install --no-install-recommends -y \
            gcc-aarch64-linux-gnu libc6-dev-arm64-cross; \
    elif [ "$TARGETARCH" = "arm" ]; then \
        apt-get install --no-install-recommends -y \
            gcc-arm-linux-gnueabihf libc6-dev-armhf-cross; \
    fi && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /code

COPY go.mod ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

FROM toolchain AS builder

ARG TARGETARCH=arm64
ARG TARGETOS=linux

ENV CGO_ENABLED=1
ENV GOOS=${TARGETOS}
ENV GOARCH=${TARGETARCH}

# Set cross-compiler based on architecture
ENV CC_arm64=aarch64-linux-gnu-gcc
ENV CC_arm=arm-linux-gnueabihf-gcc

COPY . .

RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    if [ "$TARGETARCH" = "arm64" ]; then \
        export CC=$CC_arm64; \
    else \
        export CC=$CC_arm GOARM=7; \
    fi && \
    mkdir -p /tmp/bin && \
    go build -trimpath -ldflags="-s -w" -o /tmp/bin/hello ./cmd/hello

FROM scratch AS output
COPY --from=builder /tmp/bin/hello /
```

Directory layout:

```text
.
├── Dockerfile.hello
├── cmd
│   └── hello
│       └── main.go
├── go.mod
└── pkg
    └── hello
        ├── hello.c
        ├── hello.go
        └── hello.h
```

## Build and Export

Build for 64-bit ARM (`linux/arm64`):

```bash
docker buildx build \
  -f Dockerfile.hello \
  --build-arg TARGETARCH=arm64 \
  --output type=local,dest=./bin/arm64 \
  .
```

Or build for 32-bit ARMv7 (`linux/arm`):

```bash
docker buildx build \
  -f Dockerfile.hello \
  --build-arg TARGETARCH=arm \
  --output type=local,dest=./bin/arm \
  .
```

Inspect the compiled binary:

```bash
file ./bin/arm64/hello
# ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked ...
```

## Testing Locally with QEMU

If you are running on a Linux host (or inside a container with `qemu-user`), you can execute the target binary directly through user-mode emulation without copying to physical hardware:

```bash
qemu-aarch64 -L /usr/aarch64-linux-gnu ./bin/arm64/hello Roland
# Hello, Roland!
```

Source code and additional configurations are available at [rolandjitsu/go-cross](https://github.com/rolandjitsu/go-cross).
