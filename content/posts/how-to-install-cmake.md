---
title: "How to install cmake?"
description: "Install cmake on Debian, Ubuntu or macOS."
date: 2020-09-29T14:32:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["cmake", "build-tools"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: true
---

[CMake](https://cmake.org/) is a cross-platform tool for building, testing and packaging software.

If you've been writing C or C++ (or Fortran) you probably had to, at some point, use cmake to package your software.

## Install on macOS

On macOS, the recommended method is using [Homebrew](https://brew.sh/):

```bash
brew install cmake
```

Or download the macOS `.dmg` installer directly from the [CMake download page](https://cmake.org/download/).

## Install on Linux

If you're on Debian or Ubuntu, you would normally run:

```bash
sudo apt-get install -y cmake
```

However, distros often ship older releases. Check your current version first:

```bash
cmake --version
```

If you need a newer version, you can either install from the [Kitware APT repository](https://apt.kitware.com/) (recommended for Debian/Ubuntu) or download precompiled binaries / build from source.

### Option A: Install via precompiled binaries (Fastest)

Remove any outdated package:

```bash
sudo apt-get -y purge --auto-remove cmake
```

Pick the target release from the [GitHub releases page](https://github.com/Kitware/CMake/releases) (e.g. `3.30.5`) and download the precompiled binary script:

```bash
CMAKE_VERSION="3.30.5"
ARCH="$(uname -m)" # x86_64 or aarch64
wget "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}-linux-${ARCH}.sh"
sudo sh "cmake-${CMAKE_VERSION}-linux-${ARCH}.sh" --prefix=/usr/local --skip-license
rm "cmake-${CMAKE_VERSION}-linux-${ARCH}.sh"
```

### Option B: Build from source

If you need custom build flags or want to build manually from source:

Install build dependencies:

```bash
sudo apt-get update && sudo apt-get install -y build-essential libssl-dev wget
```

Download and extract the source archive:

```bash
CMAKE_VERSION="3.30.5"
wget "https://github.com/Kitware/CMake/releases/download/v${CMAKE_VERSION}/cmake-${CMAKE_VERSION}.tar.gz"
tar -xzvf "cmake-${CMAKE_VERSION}.tar.gz"
cd "cmake-${CMAKE_VERSION}"
```

Build and install:

```bash
./bootstrap
make -j$(nproc)
sudo make install
```

## Verify the installation

```bash
cmake --version
```

You should see output similar to:

```text
cmake version 3.30.5
CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

For other platforms, check the [official install guide](https://cmake.org/install/).
