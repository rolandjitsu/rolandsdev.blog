---
title: "How to install cmake?"
description: "Install cmake on Debian, Ubuntu or macOS."
date: 2020-09-29T14:32:00+08:00
lastmod: 2022-03-19T21:34:00+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["cmake", "build-tools"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: no
---

[CMake](https://cmake.org/) is a cross-platform tool for building, testing and packaging software.

If you've been writing C or C++ (or maybe Fortran) you probably had to, at some point, use cmake to package your software.

If you're a Linux user, you'd normally just need to run:
```bash
sudo apt-get install -y cmake
```
to install it. But sometimes, the version that you can install from the official repository is not the one you need.

Therefore, you will probably need to proceed with a manual installation. But before you do that, check the current version to see whether is the one you expect or not:
```bash
cmake --version
```

**NOTE**: These instructions should work for Debian based distros and macOS.

Remove the current install:
```bash
sudo apt-get -y purge --auto-remove cmake
```

Download the [latest stable release](https://cmake.org/download), e.g [v3.19.2](https://github.com/Kitware/CMake/releases/download/v3.19.2/cmake-3.19.2.tar.gz) (do not download the binary/dmg distribution, but the Linux/Unix source):
```bash
wget https://github.com/Kitware/CMake/releases/download/v3.19.2/cmake-3.19.2.tar.gz
```

Unzip the archive:
```bash
tar -xzvf cmake-3.16.4.tar.gz
```

Build and install it:
```bash
cd cmake-3.16.4
./bootstrap
make -j$(nproc)
sudo make install
```

Verify the installation:
```bash
cmake --version
```

If you see something like:
```text
cmake version 3.19.2
CMake suite maintained and supported by Kitware (kitware.com/cmake).
```
you're good to go.

For other platforms, check the [official install guide](https://cmake.org/install/).
