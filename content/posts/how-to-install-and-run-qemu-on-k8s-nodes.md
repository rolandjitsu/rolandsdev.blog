---
title: "How to install and run QEMU on K8s nodes?"
description: "Add support for multi-CPU architectures on Kubernetes nodes."
date: 2020-10-04T14:57:00+08:00
lastmod: 2026-11-17T17:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["k8s", "qemu"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: true
---

If you find yourself having to emulate different CPU architectures in a Kubernetes environment you'll probably end up running some version of [binfmt](https://github.com/tonistiigi/binfmt) as an [init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) or maybe manually run it once.

That would probably be ok in most cases, but wouldn't work if, for example, you're running multiple pod replicas on the same node concurrently (say, when you setup a CI that spawns pods for every job in your pipeline) or when nodes are autoscaled.

There're ways to make sure only one pod sets up binfmt, but when running pods concurrently, it's very difficult to orchestrate.

The cleaner solution is a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), which runs binfmt once on any existing or newly scheduled node in your cluster.

Create a daemonset configuration:

```yaml
# Run binfmt setup on any node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: binfmt
  namespace: kube-system
  labels:
    app: binfmt-setup
spec:
  selector:
    matchLabels:
      name: binfmt
  template:
    metadata:
      labels:
        name: binfmt
    spec:
      tolerations:
        # Allow daemonset to run on control plane nodes if needed
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      initContainers:
        - name: binfmt
          image: tonistiigi/binfmt:latest
          args: ["--install", "all"]
          securityContext:
            privileged: true
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            limits:
              cpu: 50m
              memory: 50Mi
            requests:
              cpu: 10m
              memory: 20Mi
```

Apply it:

```bash
kubectl apply -f ./daemonset.yaml
```

Once the pods transition to running, QEMU and `binfmt_misc` handlers are registered in the host Linux kernel for all nodes.

{{< admonition type=note >}}
If you decide on a different approach later, remove the daemonset and its pods:

```bash
kubectl delete daemonset binfmt --namespace=kube-system
```
{{< /admonition >}}
