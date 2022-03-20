---
title: "Build Docker images on k8s nodes"
description: "Speed up your Docker builds using buildx and Kubernetes."
date: 2020-10-10T17:19:00+08:00
lastmod: 2022-03-20T19:35:15+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["k8s", "qemu"]
categories: ["DevOps"]
series: []

toc:
  enable: no
---

In one of my previous posts, [cross-compiling for Raspberry Pi with Docker]({{< ref "/posts/cross-compile-for-raspberry-pi-with-docker" >}}), I wrote about and illustrated how the relatively new [buildx](https://github.com/docker/buildx) command made it much easier to build and dump binaries on the host system.

Another useful feature of buildx is the ability to use different drivers when building images. As of today (10.10.2020), it supports [3 different drivers](https://github.com/docker/buildx/tree/d1a46faf8475564fbb27c9228b79a806e59b8f65#--driver-driver):
1. `docker` - uses the docker daemon built in builder (default). You'd use this if you want to build and use images locally.
2. `docker-container` - uses a [BuildKit](https://github.com/moby/buildkit) container spawned by docker. Useful when you want to build multi-platform images.
3. `kubernetes` - uses k8s pods with defined buildkit container images. Similar to the `docker-container` driver, but instead of building on the host, it builds on a k8s node

I find the `kubernetes` driver quite useful when you're low on resources on your host or when you just want your builds to be fast; there's probably other good reasons to use it (scalability?), but I'm not going to try to convince you. You should just try it and see if it fits your needs or not.

And the setup doesn't take more than a minute, definitely less than the time it takes to read this post.

I'm going to assume you've already created a K8s cluster on [GCP](https://cloud.google.com/kubernetes-engine/docs/quickstart#create_cluster) or [AWS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) (or some other cloud provider), but to make sure, check that the current context is what you expect:
```bash
kubectl config current-context
```

Should, on a GKE cluster, show something similar to:
```text
gke_<project name>_<region>_<cluster name>
```

Now, create a builder:
```bash
docker buildx create --name <builder name> --driver kubernetes --use
```

You can provide other [options](https://github.com/docker/buildx/tree/d1a46faf8475564fbb27c9228b79a806e59b8f65#--driver-opt-options) too; let's say you want a couple of replicas:
```bash
docker buildx create --name <builder name> \
  --driver kubernetes \
  --driver-opt replicas=2\
  --use
```
{{< admonition type=info >}}
The `--use` flag will switch to that builder automatically.
{{< /admonition >}}

And bootstrap it:
```bash
docker buildx inspect --bootstrap
```

That's it, you're ready to build images on your k8s nodes.
