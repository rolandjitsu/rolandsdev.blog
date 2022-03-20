---
title: "Install and run qemu on K8s nodes"
description: "Add support for multi-CPU architectures on Kubernetes nodes."
date: 2020-10-04T14:57:00+08:00
lastmod: 2022-03-20T19:16:57+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["k8s", "qemu"]
categories: ["DevOps"]
series: ["How to do stuff"]

toc:
  enable: no
---

If you find yourself having to emulate different CPU architectures in a Kubernetes environment you'll probably end up running some version of [binfmt](https://hub.docker.com/r/linuxkit/binfmt) as an [init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) or maybe manually run it once.

That would probably be ok in most cases, but wouldn't work if, for example, you're running multiple pod replicas on the same node concurrently (say, when you setup a CI that spawns pods for every job in your pipeline) or when nodes are autoscaled.

There're probably ways to make sure only one pod sets up binfmt, but when running the pods concurrently, it's very difficult to orchestrate.

But there's light at the end of the tunnel. There's the [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) which we can use to run binfmt once on any newly created node.

You just need to create a simple config with:
```yaml
# Run binfmt setup on any new node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: binfmt
  labels:
    app: binfmt-setup
spec:
  selector:
    matchLabels:
      name: binfmt
  # https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates
  template:
    metadata:
      labels:
        name: binfmt
    spec:
      tolerations:
        # Have the daemonset runnable on master nodes
        # NOTE: Remove it if your masters can't run pods
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      initContainers:
        - name: binfmt
          image: tonistiigi/binfmt
          # command: []
          args: ["--install", "all"]
          # Run the container with the privileged flag
          # https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
          # https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#securitycontext-v1-core
          securityContext:
            privileged: true
      containers:
        - name: pause
          image: gcr.io/google_containers/pause
          resources:
            limits:
              cpu: 50m
              memory: 50Mi
            requests:
              cpu: 50m
              memory: 50Mi
```

And apply it:
```bash
kubectl apply -f ./daemonset.yaml
```

That's it. binfmt should now be setup on your cluster.

{{< admonition type=note >}}
If later on you decide on a different approach, you can delete the daemonset and any resources associated with it:
```bash
kubectl delete daemonset binfmt --namespace=default
```
{{< /admonition >}}
