---
title: "Build Docker images on k8s nodes"
description: "Speed up your Docker builds using buildx and Kubernetes."
date: 2020-10-10T17:19:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["k8s", "docker", "buildx"]
categories: ["DevOps"]
series: []

toc:
  enable: true
---

[Docker Buildx](https://docs.docker.com/build/buildx/) allows you to route image build execution away from your local workstation using pluggable [builder drivers](https://docs.docker.com/build/builders/drivers/):

1. **`docker`**: Uses the standard daemon engine (default). Best for single-arch, local image generation.
2. **`docker-container`**: Spawns an isolated BuildKit container locally. Supports multi-platform builds and cache export.
3. **`kubernetes`**: Deploys BuildKit pods dynamically into a Kubernetes cluster, offloading compilation, packaging, and disk I/O to remote node pools.
4. **`remote`**: Directly connects to a standalone BuildKit daemon over mTLS or TCP, useful for dedicated shared build infrastructure.

The `kubernetes` driver is particularly effective when local CPU and RAM are constrained, or when you want your build tasks to scale horizontally across high-performance cloud compute.

## Prerequisites

Verify your `kubectl` context points to the target cluster:

```bash
kubectl config current-context
```

## Creating the Kubernetes Builder

Create a new Buildx builder using the `kubernetes` driver:

```bash
docker buildx create \
  --name k8s-builder \
  --driver kubernetes \
  --driver-opt namespace=buildkit,replicas=2 \
  --driver-opt requests.cpu=2,requests.memory=4Gi \
  --driver-opt limits.cpu=4,limits.memory=8Gi \
  --use
```

{{< admonition type=info >}}
The `--use` flag automatically switches your current Buildx context to this new builder.
{{< /admonition >}}

Bootstrap the builder instance to pull the BuildKit image and launch the Kubernetes pods:

```bash
docker buildx inspect --bootstrap
```

Check that the builder pods are running in your cluster:

```bash
kubectl get pods -n buildkit -l app=buildkit
```

## Running Builds with Remote Caching

Once initialized, run `docker buildx build` as usual. Because the builder lives inside the cluster, push outputs to a container registry or export them locally:

```bash
# Push directly to registry with inline cache
docker buildx build \
  --tag my-registry.example.com/app:v1 \
  --push \
  --cache-to type=registry,ref=my-registry.example.com/app:buildcache,mode=max \
  --cache-from type=registry,ref=my-registry.example.com/app:buildcache \
  .
```

Or export the resulting filesystem directly back to your local machine:

```bash
docker buildx build -o type=local,dest=./dist .
```

## Cleanup

To remove the builder and its Kubernetes resources:

```bash
docker buildx rm k8s-builder
```
