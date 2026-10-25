---
title: "Replacing our CI registry with Harbor on Garage"
description: "Part 1 of a series: why deploy a new registry, the Harbor + Garage + Longhorn design, a replication-factor lesson, and how faster pulls broke the k8s apiserver."
date: 2026-10-18T16:00:00+09:00
lastmod: 2026-10-25T16:30:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["kubernetes", "harbor", "garage", "ci", "registry"]
categories: ["DevOps"]
series: ["an oci registry for ci"]

toc:
  enable: true
---

Over the past month and a half I've been rolling out a new OCI registry for our CI: [Harbor](https://goharbor.io) as the registry, [Garage](https://garagehq.deuxfleurs.fr) as the S3 storage behind it, on Kubernetes. It was quite the journey - a couple of incidents, a lot of measuring, and a pile of lessons about what actually limits a registry under CI load.

This is part 1: why we built it, the design, and the things that broke before it even carried real traffic. Part 2 will be the throughput hunt (every layer we fixed, and the surprising number that didn't improve). Part 3 is the storage war - including the weekend I already wrote about in [the fstrim post]({{< ref "/posts/reclaim-deleted-disk-space-with-fstrim-in-longhorn" >}}).

## Why a new registry

The old registry had served us well for years, but CI outgrew it. Under load, large-image pushes would fail with 502/503/504, and pulls would stall - at the worst moments throughput dropped to single-digit Mbit/s. Some stats measured over a week: **12.2% of all image pulls took more than 5 minutes**, 6% more than 10 minutes. A slow pull wasn't just slow - a job that stalled pulling its image ran out its 1-hour timeout and got killed, wasting a full compute-hour. When we classified failed CI jobs by cause, roughly **1 in 5 traced back to the registry**.

So the goals for the replacement, roughly in order:

1. **Stability and scalability** - survive node failures, scale out by adding nodes.
2. **Throughput** - CI fans out hundreds of concurrent pulls; the registry should not be the bottleneck.
3. **Retention control** - CI churns through enormous numbers of short-lived images; we wanted real retention policies and GC instead of custom cleanup scripts - which sometimes removed blobs still shared by images we needed.
4. **Observability and control** - dashboards, metrics, and insight into every component, storage included. The old registry had metrics, but they lived with the infra team; consolidating observability into one place we own - so we can monitor and alert on issues ourselves - is a goal of its own, and the registry should be part of it.

## The design

[Harbor](https://goharbor.io) for the registry itself (auth, projects, retention, GC, a UI people can actually use), backed by [Garage](https://garagehq.deuxfleurs.fr) - a self-hosted, S3-compatible object store that replicates data across its own nodes. Garage was our infra team's suggestion when we asked what we could run instead of MinIO, which is no longer open source. And it fit: it's lightweight, runs well on Kubernetes, and its replication model is simple to reason about - every object is stored on `replication_factor` nodes, no erasure coding, no external coordinator. The honest trade-off: Garage optimizes for simplicity and resilience over raw speed - its own documentation describes throughput as moderate compared to MinIO or Ceph. We'll get back to that in part 2.

The request path, as first deployed:

{{< mermaid >}}
flowchart LR
    c["container runtime (CI node)"] -->|pulls + pushes, TLS| t["Traefik (ingress)"]
    t --> h["Harbor core + registry"]
    h -->|"S3 API"| g[("Garage, 3 nodes")]
{{< /mermaid >}}

Everything - manifests, auth, and every blob byte - flowed through Traefik and Harbor's registry process. Harbor was deliberately set to *proxy* blobs rather than redirect clients to S3, since Garage was internal-only at that point. Keep that detail in mind for part 2.

And the storage stack below Garage, which is where a lot of this series happens:

{{< mermaid >}}
flowchart TB
    g["Garage pod (one per storage node)"] --> pv["Longhorn PV (strict-local, 1 replica)"]
    pv --> vd["VM virtual disk (virtio)"]
    vd --> px["Proxmox storage layer (Ceph, over the network)"]
    px --> d[("physical disks, somewhere in a rack")]
{{< /mermaid >}}

One deliberate decision here: the Longhorn StorageClass for Garage uses `numberOfReplicas: 1` with `dataLocality: strict-local`. Garage already replicates every object across nodes at the application level; if Longhorn also replicated each volume, we'd multiply the two factors and pay for the same data many times over, in both disk and I/O. So Longhorn keeps a single local copy and Garage owns durability.

I was quite pleased with that reasoning. What I did not fully register at the time: four of the five storage nodes were Proxmox VMs whose virtual disks sat on Ceph - network-backed storage with replication machinery of its own (whether and how it replicates in our setup is infra's domain - I honestly don't know). We had carefully avoided replication stacking at the layer we controlled - and never looked at the layers below it. That, plus what virtualized storage does to latency, is the villain of part 3.

## Day 2: the 2GB push cliff

The first real bug showed up a day after deploying. Pushes of large images kept failing the same way: the layer upload would stop at around 2GB, then retry from zero, loop forever.

The cause was a Traefik `buffering` middleware on the registry route - added, ironically, *to help* large pushes by enabling retries :roll_eyes: . What we missed is that for Traefik to be able to retry a request, it has to buffer the **whole request body**, spilling to a temp file, and that spill area had a ~2GB ceiling. Anything larger: connection reset, and the client retries the layer from byte zero.

The fix was to delete the middleware and let `/v2/` stream. The lesson is broader than Traefik: **a reverse proxy that buffers requests has a size limit somewhere, and container layers will find it.**

## Replication factor is not failure tolerance

When I set up Garage I had to pick a replication factor, and I picked 2. Three nodes, every object on two of them, raw capacity divided by two instead of three - "tolerates 1 node failure", said the note next to the setting :expressionless: . More than once I caught myself thinking *isn't RF=2 good enough?*

Five days in, one of the three storage nodes went NotReady, and about **two thirds of all objects became unreadable**. Image pulls 500'd, Harbor's UI threw errors, and CI - whose executor images we had just moved to the new registry - started failing. The images were never deleted, nothing was lost; they were simply *unavailable*, because with `replication_factor = 2` Garage needs a quorum of 2 - both copies - for consistent operations. One node down means every object with a copy on that node is out of reach.

**RF=2 tolerates zero node failures.** RF=3 is the minimum that tolerates one, because 2 of 3 copies still form a quorum. My intuition said that a second copy means you can afford to lose a node and still serve the data; for operations that need a quorum, it does not - and the setting's own description agreed with my wrong intuition.

Changing it was its own small adventure: Garage persists the replication factor inside the cluster layout on every node's metadata volume and refuses to start when the config disagrees with what's on disk. With only ~28GB per node stored at that point, the pragmatic fix was to stop the cluster, delete the persisted layout, and let it rebuild and rebalance - about 15 minutes. On a full cluster, that would have been a very different day. **Pick your replication factor like it's permanent, because in practice it is.** The same goes for anything else that pins state down - PVs tied to specific nodes, storage classes, data locality settings: all of it is a pain to change once live data sits on top.

## The sizing math

Storage sizing looked straightforward. From discussions with the team running the old registry: it churned through about **12TB a week**, roughly half of which their cleanup reclaimed by week's end - so ~6TB a week sticking around under their retention rules. Our retention would run on a different schedule: most CI-generated images are short-lived (only stable images stay forever), so instead of a weekly cleanup I planned retention + GC every other day, and figured ~2TB of usable space buys a couple of days of inflow - we'd oscillate in a safe band. We started with 3 nodes x 500GB and grew to 5 nodes x 800GB as they became available - about 1.3TB usable at RF=3.

The churn estimate was in the right ballpark - measured later, CI generated 0.5-1TB of image garbage per day while the new registry carried only part of the load. The capacity math still blew up in our faces, three different ways:

- The plan depended on GC *running and actually reclaiming space* from day one. We deployed with Harbor's GC bootstrap intentionally disabled - one less moving part during the rollout - and retention policies came later and tighter than planned.
- "Usable space" assumed deleting data frees disk. Below a thin-provisioned block layer, [it does not]({{< ref "/posts/reclaim-deleted-disk-space-with-fstrim-in-longhorn" >}}).
- Growing the volumes under a live, replicated store turned out to be an operation with its own failure modes.

Each of those is a later chapter. The takeaway for part 1 is just this: **sizing a registry is not about average inflow, it's about the reclamation loop** - how fast garbage is created versus how reliably the whole delete-GC-reclaim chain keeps up. Averages said we were fine; the loop said otherwise.

## Fast pulls broke the apiserver

Here's the side effect I didn't see coming at all.

Our main CI runner was configured with `concurrent: 306`. On the old registry this number was mostly theoretical - jobs trickled in as slow pulls staggered them. On the new registry, pulls were fast, so when a big pipeline landed (one schedule alone fans out to **150+ pull-heavy jobs, concurrently**), the runner would try to start all of them *at once*. And a runner job is not one API request: the [Kubernetes executor](https://docs.gitlab.com/runner/executors/kubernetes/) creates a pod per job (a build container, a helper, one per service) and then drives each container through the API - attach/exec to run the scripts, log streaming, status watches. Multiply by 150. Faster storage turned a gentle ramp into a stampede :sweat_smile: .

The stampede hit in layers:

- First the kubelet: the default image-pull rate limit (`registry-pull-qps=5`) rejected pulls -> `ErrImagePull: pull QPS exceeded` -> GitLab retried -> more churn. Fixed with `registry-qps=0` (unlimited) on the CI node kubelets - the quick/easy option; there was headroom to simply raise the limit, and unlimited is probably not the wisest setting - plus `max-concurrent-downloads` 10 -> 128 for the container runtime.
- Then the apiserver: runner jobs failed with `starting pod watcher: not synced: *v1.Pod`. The runner's pod creates and informer LIST/WATCH requests were landing in the default `service-accounts` [API Priority and Fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/) flow schema, whose queue filled up - the apiserver answered with HTTP 429, over **5,000 rejections on a single apiserver** during one burst. The control plane itself was nearly idle (~12% CPU) - this was pure admission control, not capacity.

So we gave the runners their own APF lane: a dedicated FlowSchema and PriorityLevelConfiguration, so runner traffic queues separately from everything else instead of competing in the shared lane. That stopped the 429s.

Two weeks later the error came back anyway, and this one taught me how APF actually behaves when a lane is undersized: with a queueing priority level there are **no rejections** - over-limit requests just *wait*. Our lane had ~100 seats per apiserver; the burst asked for **989**. Worse, 989 was bigger than the apiserver's *entire* default request budget (`max-requests-inflight` + `max-mutating-requests-inflight` = 400 + 200). No amount of APF tuning can queue your way out of demand that exceeds the whole server's ceiling - the queue wait just exceeds the client's timeout, and the user-visible error is identical to the 429 it replaced.

And one more subtlety: a runner is a single client-go process, which means a single HTTP/2 connection, which means **all of its ~970 pod watches pin to one apiserver**. Load balancing across the control plane does nothing for one fat client.

The final shape:

- **Split the runner into 3 replicas** (`concurrent: 111` each) so the watch load spreads across apiservers.
- **Raised the apiserver budget**: `max-requests-inflight` 400 -> 1400, `max-mutating-requests-inflight` 200 -> 600 - sized so a *single* apiserver can absorb the whole burst, because nothing guarantees the three runner connections land on distinct apiservers.
- **Resized the APF lane** to ~1100 seats per apiserver, above the measured 989-seat peak.

**If you make the storage faster, you move the bottleneck up the stack.** The apiserver had always been protected by the old registry's slowness. Nobody had ever asked it to start 306 pods in the same second before.

## Where that left us

By the end of September the new registry was quietly carrying a small slice of CI - at one point I counted 1,233 containers pulling from the old registry and 4 from the new one - and then we started flipping pipeline schedules over, including the hourly one with the 150+ concurrent pulls. Dashboards were green. A live pipeline pulling 10-14GB images peaked the storage at ~105 MB/s - under 10% of the aggregate ceiling we'd measured. The registry was not the bottleneck, and for a while it genuinely wasn't.

What I took away from the rollout phase:

1. **Replication factor is not failure tolerance** - RF=2 tolerates zero node failures. Treat RF - and anything else that pins data to nodes or storage classes - as permanent.
2. **A buffering reverse proxy has a size cliff somewhere**; large container layers will find it.
3. **Sizing is about the reclamation loop**, not average inflow.
4. **Every speedup moves the bottleneck somewhere else** - ours moved into the Kubernetes control plane.

Part 2 is the throughput hunt: why blobs stopped flowing through the registry entirely, hardware crypto acceleration that was advertised but not actually working, a flannel behavior we didn't know about, and how after fixing all of it the single-stream number barely improved - for a reason that was in Garage's documentation all along.
