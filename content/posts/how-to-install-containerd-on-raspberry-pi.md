---
title: "How to install containerd on Raspberry Pi"
description: "Install containerd on Raspberry Pi in 2 minutes."
date: 2020-12-06T15:52:00+08:00
lastmod: 2026-11-17T14:30:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["raspberry-pi", "containerd", "runc", "go"]
categories: ["DevOps", "Raspberry Pi"]
series: ["How to do stuff"]

toc:
  enable: true
---

You should be running Raspberry Pi OS based on Debian Bookworm (Debian 12) or Bullseye. Modern Raspberry Pi models (Pi 3, 4, 5, Zero 2 W) can run either 64-bit (`arm64`) or 32-bit (`armhf`) OS images, while older models (Pi 1, 2, Zero) run 32-bit (`armhf`/`armv6l`).

Before starting, update your package lists and installed packages:

```bash
sudo apt update
sudo apt full-upgrade -y
```

## Install Go

We need Go to compile [runc](https://github.com/opencontainers/runc) and containerd.

Download and extract the Go archive matching your Pi architecture (check [Go downloads](https://go.dev/dl/) for other versions):

### For 64-bit Raspberry Pi OS (aarch64 / arm64)

```bash
wget https://go.dev/dl/go1.23.6.linux-arm64.tar.gz
sudo tar -C /usr/local -xzf go1.23.6.linux-arm64.tar.gz
rm go1.23.6.linux-arm64.tar.gz
```

### For 32-bit Raspberry Pi OS (armhf / armv6l)

```bash
wget https://go.dev/dl/go1.23.6.linux-armv6l.tar.gz
sudo tar -C /usr/local -xzf go1.23.6.linux-armv6l.tar.gz
rm go1.23.6.linux-armv6l.tar.gz
```

Add the Go binary directory to your `PATH` (in `~/.profile` or `~/.bashrc`):

```bash
echo 'export PATH="$PATH:/usr/local/go/bin"' | tee -a $HOME/.profile
source $HOME/.profile
```

Verify the installation:

```bash
go version
# Example: go version go1.23.6 linux/arm64
```

## Install runc

`runc` is the OCI runtime used by containerd to spawn and manage container processes.

Install build dependencies and `libseccomp` development headers:

```bash
sudo apt install -y git make gcc pkg-config libseccomp-dev
```

Clone and compile `runc`:

```bash
git clone https://github.com/opencontainers/runc.git
cd runc
make -j$(nproc)
sudo make install
cd ..
```

Verify the installation:

```bash
runc --version
```

## Install containerd

Install the required storage libraries and tools:

```bash
sudo apt install -y btrfs-progs libbtrfs-dev protobuf-compiler
```

Clone the `containerd` repository and check out a release tag (such as `v2.0.2` for containerd v2, or `v1.7.25` for containerd v1 LTS):

```bash
git clone https://github.com/containerd/containerd.git
cd containerd
git checkout v2.0.2
make -j$(nproc)
sudo make install
```

Verify the installation:

```bash
containerd --version
# Example: containerd github.com/containerd/containerd v2.0.2 ...
```

## Configure containerd

Generate a default containerd configuration file at `/etc/containerd/config.toml`:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

## Run containerd as a systemd service

Install and start the systemd unit file from the containerd repository:

```bash
sudo cp containerd.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd.service
```

Check the status to ensure it is active and running:

```bash
sudo systemctl status containerd.service
```

## Testing containerd

Test the containerd client CLI (`ctr`):

```bash
sudo ctr version
```

To test running containers with local OCI images without a registry, see the companion repository [rolandjitsu/containerd-oci-import](https://github.com/rolandjitsu/containerd-oci-import). It demonstrates how to build and export OCI tar archives using Docker Buildx and interact with containerd programmatically via its Go client.
