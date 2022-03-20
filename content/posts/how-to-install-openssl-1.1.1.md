---
title: "How to install OpenSSL 1.1.1?"
description: "Install OpenSSL from source on Debian, Ubuntu or macOS."
date: 2020-09-27T16:47:00+08:00
lastmod: 2022-03-19T22:27:42+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["openssl"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: yes
---

[OpenSSL](https://github.com/openssl/openssl) is a TLS/SSL and crypto library.

Most Linux distributions come packaged with some older version of OpenSSL, but if you need some of newest features (such as [support for TLSv1.3](https://www.openssl.org/news/openssl-1.1.1-notes.html)), then you'll need to manually install it.

## Install on Linux
These instructions should work for most Debian based distros.

Install the build dependencies:
```bash
sudo apt-get -y install build-essential checkinstall git zlib1g-dev
```

Clone OpenSSL:
```bash
git clone --depth 1 --branch OpenSSL_1_1_1g https://github.com/openssl/openssl.git
```

Configure it:
```bash
cd openssl
./config zlib '-Wl,-rpath,$(LIBRPATH)'
```

{{< admonition type=note >}}
The rpath flag is used to set the runtime shared library search path, check the notes about [shared libs and non-default install locations](https://github.com/openssl/openssl/blob/master/NOTES-Unix.md#shared-libraries-and-installation-in-non-default-locations).
{{< /admonition >}}

Build and test:
```bash
make
make test
```

Install it:
```bash
sudo make install
```

And finally, configure the shared libs:
```bash
sudo ldconfig -v
```

If you run:
```bash
openssl version
```

you should see the following output:
```text
OpenSSL 1.1.1g  21 Apr 2020
```

## Install on macOS
Use [brew](https://brew.sh/) to install it:
```bash
brew install openssl@1.1
```

You might also need to add the binary to the PATH:
```bash
echo 'export PATH="/usr/local/opt/openssl@1.1/bin:$PATH"' >> ~/.bash_profile
```

Verify the install:
```bash
openssl version
```

That's it.
