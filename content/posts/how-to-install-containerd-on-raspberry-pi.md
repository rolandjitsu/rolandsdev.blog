---
title: "How to install containerd on Raspberry Pi"
description: "Install containerd on Raspberry Pi in 2 minutes."
date: 2020-12-06T15:52:00+08:00
lastmod: 2022-03-20T21:24:44+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["raspberry-pi", "containerd", "runc", "go"]
categories: ["DevOps", "Raspberry Pi"]
series: ["How to do stuff"]

toc:
  enable: yes
---

You need to be running at least Raspbian OS with Buster or Raspberry Pi OS (might work with older OS versions too, but I haven't tested it).

I've also updated the system, firmware, apps, etc to the latest version:
```bash
sudo apt update
sudo apt full-upgrade
sudo rpi-update
```

## Install Go
We need Go to install [runc](https://github.com/opencontainers/runc) and containerd.

Download the Go archive (check [downloads](https://golang.org/dl/) for other distributions):
```bash
wget https://golang.org/dl/go1.15.6.linux-armv6l.tar.gz
```

Extract the archive to `/usr/local`:
```bash
tar -C /usr/local -xzf go1.15.6.linux-armv6l.tar.gz
```

Add the binary to `PATH` (in `~/.profile` or `~/.bash_profile`):
```bash
echo 'PATH="$PATH:/usr/local/go/bin"' | tee -a $HOME/.profile
```

Set the `GOPATH` (get it from `go env GOPATH`):
```bash
echo "export GOPATH=$(go env GOPATH)" | tee -a $HOME/.profile
```

Reload the shell and verify the installation:
```bash
source $HOME/.profile
go version
# go version go1.15.6 linux/arm
```

## Install runc
We need runc to spawn and run containers.

Install `libseccomp-dev` for `seccomp` support:
```bash
sudo apt install libseccomp libseccomp-dev
```

[Install runc](https://github.com/containerd/containerd/blob/master/BUILDING.md#build-runc):
```bash
go get -v github.com/opencontainers/runc
cd $GOPATH/src/github.com/opencontainers/runc
make -j`nproc`
sudo env PATH="$PATH" make install
```

## Install containerd
Install btrfs libs and tools:
```bash
sudo apt install btrfs-progs libbtrfs-dev
```

Install the tools we need to build protobuf (we assume `make`, `g++`, `curl` and `unzip` are already installed):
```bash
sudo apt install autoconf automake libtool
```

Install [protobuf](https://github.com/protocolbuffers/protobuf/blob/master/src/README.md) (if g++ dies or ld errors, lower the cpu cores used, e.g `-j3`, `-j2`, `-j1` and/or [enable overclocking](https://www.raspberrypi.org/documentation/configuration/config-txt/overclocking.md) and [increase swap](https://pimylifeup.com/raspberry-pi-swap-file/)):
```bash
git clone https://github.com/protocolbuffers/protobuf.git
cd protobuf
git checkout v3.14.0
git submodule update --init --recursive
./autogen.sh
./configure
make -j2
make -l2 -j2 check
sudo make install
sudo ldconfig
```

{{< admonition type=note >}}
If the `IoTest.LargeOutput` test fails, it's probably just due to the lack of memory.
{{< /admonition >}}

[Install containerd](https://github.com/containerd/containerd/blob/master/BUILDING.md):
```bash
go get -v github.com/containerd/containerd
cd $GOPATH/src/github.com/containerd/containerd
make -j2
sudo env PATH="$PATH" make install
```

Verify the installation:
```bash
containerd --version
# containerd github.com/containerd/containerd v1.4.0-2397-ge98d7f8ea e98d7f8eaafce9da741ef37c7fedcde571806dcb
```

Run containerd as a systemd service:
```bash
cd $GOPATH/src/github.com/containerd/containerd
sudo cp containerd.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable containerd.service
sudo systemctl start containerd.service
```

That's it, containerd should be up and running.

To test it, follow the [getting started guide](https://containerd.io/docs/getting-started/), or follow the simple guide from [rolandjitsu/containerd-oci-import](https://github.com/rolandjitsu/containerd-oci-import) (illustrates how to use local OCI images too).
