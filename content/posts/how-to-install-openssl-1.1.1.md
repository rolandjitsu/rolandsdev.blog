---
title: "How to install OpenSSL 1.1.1?"
description: "Install OpenSSL from source on Debian, Ubuntu or macOS."
date: 2020-09-27T16:47:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["openssl"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: true
---

{{< admonition type=danger title="OpenSSL 1.1.1 is End-of-Life" >}}
**OpenSSL 1.1.1 reached official End of Life (EOL) on September 11, 2023.** It no longer receives security vulnerability fixes or patches.

For modern deployments, install **OpenSSL 3.x** (the current standard LTS branch). Modern Linux distributions (Ubuntu 22.04+, Debian 12+) already ship with OpenSSL 3.x by default (`sudo apt-get install -y openssl libssl-dev`).
{{< /admonition >}}

[OpenSSL](https://github.com/openssl/openssl) is a cryptography toolkit and TLS/SSL library.

## Modern installation: OpenSSL 3.x

### On macOS

Install OpenSSL 3 using [Homebrew](https://brew.sh/):

```bash
brew install openssl@3
```

Homebrew installs packages to `/opt/homebrew` on Apple Silicon and `/usr/local` on Intel. Add the binaries to your shell configuration (`~/.zshrc`):

```bash
echo 'export PATH="$(brew --prefix openssl@3)/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
openssl version
```

### On Linux (Compile OpenSSL 3.x from source)

If you need a custom prefix or build flags on Debian/Ubuntu:

Install dependencies:

```bash
sudo apt-get update && sudo apt-get install -y build-essential checkinstall git zlib1g-dev
```

Clone the latest OpenSSL 3.x LTS branch (e.g. `openssl-3.4.0`):

```bash
git clone --depth 1 --branch openssl-3.4.0 https://github.com/openssl/openssl.git
cd openssl
./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl shared zlib '-Wl,-rpath,$(LIBRPATH)'
make -j$(nproc)
make test
sudo make install_sw
```

Configure shared libraries:

```bash
echo "/usr/local/ssl/lib64" | sudo tee /etc/ld.so.conf.d/openssl.conf
sudo ldconfig -v
/usr/local/ssl/bin/openssl version
```

---

## Legacy instructions: OpenSSL 1.1.1 (Historical)

{{< admonition type=note >}}
Retained only for legacy environments that cannot yet migrate to OpenSSL 3.x.
{{< /admonition >}}

Install the build dependencies:

```bash
sudo apt-get -y install build-essential checkinstall git zlib1g-dev
```

Clone OpenSSL:

```bash
git clone --depth 1 --branch OpenSSL_1_1_1w https://github.com/openssl/openssl.git
cd openssl
./config zlib '-Wl,-rpath,$(LIBRPATH)'
make
make test
sudo make install
sudo ldconfig -v
```

Verify:

```bash
openssl version
```
