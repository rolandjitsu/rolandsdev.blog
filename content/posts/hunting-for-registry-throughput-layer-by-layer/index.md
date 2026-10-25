---
title: "Hunting for registry throughput, layer by layer"
description: "Part 2 of the OCI registry series: bypassing the proxy path, hardware crypto that wasn't working, the flannel overlay cost - and the single-stream number that barely improved anyway."
date: 2026-10-25T16:30:00+09:00
lastmod: 2026-10-25T16:30:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["kubernetes", "garage", "harbor", "networking", "performance"]
categories: ["DevOps"]
series: ["an oci registry for ci"]

toc:
  enable: true
---

In [part 1]({{< ref "/posts/replacing-our-ci-registry-with-harbor-on-garage" >}}) we setup a Harbor registry backed by [Garage](https://garagehq.deuxfleurs.fr) and survived the first week. This part is about making it fast - or more precisely, about finding out, one layer at a time, what "fast" was even limited by.

TLDR: we fixed four genuinely broken or suboptimal things - the proxy path, hardware crypto, the overlay network, NIC queues - and the single-stream pull number barely improved. The reason was in Garage's documentation all along, and it turned out not to matter much. But the fixes were still worth it, for reasons that only became clear once we could measure each layer by itself.

## Measure first: where do the bytes slow down?

A week in, pulls felt slower than the storage should allow. So instead of tuning things, we ran the same test - 16 parallel downloads of a ~1.3GB blob - against each layer of the path, removing one at a time:

| Path | Aggregate | Per-stream |
|---|---:|---:|
| Garage directly (internal S3 service) | 818 MB/s | 59 MB/s |
| + Traefik (TLS ingress) | 420 MB/s | 30 MB/s |
| + Harbor core/registry proxy (the full path clients used) | 238 MB/s | **15 MB/s** |

Each proxy hop roughly halved throughput. And the per-stream number is what sets a job's pull time: **15 MB/s through the full path vs 59 MB/s straight from storage** - the difference between a 14GB image taking ~15 minutes and ~4 minutes. Neither hop was CPU-bound; the time went into proxying itself.

(A note on sizes before more numbers: the test blobs and images in this post vary - 1.3GB here, 846MB and 4.4GB later, with real CI images going up to ~14GB. They are different objects from different tests; what matters is the before/after within each test, not the sizes across tests.)

Two details from this round that paid off later:

- A dedicated TLS terminator was ~2.5x more efficient per pod than Traefik at moving big blobs.
- The TLS connections were negotiating `ChaCha20-Poly1305` instead of `AES-GCM`. I didn't really pay attention to it at the time.

## Fix #1: get the blobs out of the proxy path

A registry's control plane (auth, manifests, tags) is insignificant traffic compared to the data plane that serves blobs. There is no reason for blob bytes to flow through the ingress and the registry process - the registry's job is to say *where* the bytes are, and S3 can serve them directly.

Harbor supports exactly that: instead of proxying blobs, it answers a blob request with a **302 redirect to a presigned S3 URL**, and the client downloads straight from storage. We had initially disabled the redirect to keep the first deployment simple and ship fast; the fix was to setup a dedicated LoadBalancer IP with an HAProxy TLS terminator in front of the S3 API, and flip Harbor to redirect mode.

{{< mermaid >}}
flowchart LR
    c["container runtime (CI node)"] -- "manifests + auth (TLS)" --> t["Traefik -> Harbor"]
    c -- "blobs: 302 redirect (TLS)" --> lb["HAProxy S3 LB (dedicated LB IP)"]
    t -- "S3 API" --> g[("Garage")]
    lb --> g
{{< /mermaid >}}

A few implementation notes worth mentioning:

- **The runtime must trust the new endpoint's certificate.** Redirecting blobs to an endpoint whose issuer the container runtime does not trust fails every pull with x509 errors. We reused our existing cert issuing setup for the new endpoint - no new machinery needed.
- **Don't rewrite the Host header.** S3 request signatures include it; a proxy rewrite breaks every presigned URL.
- **HTTP/1.1 between HAProxy and clients.** S3 is HTTP/1.1, and HTTP/2 flow control can throttle large downloads through a proxy.

The result, same benchmark: **aggregate 233 -> 1026 MB/s (4.4x), per-stream 15 -> ~128 MB/s (8.5x)** :exploding_head: .

## Fix #2: Garage's block size

Single-stream pulls still maxed out around ~110 MB/s, and this one was actually Garage's: it splits objects into blocks, and the default `block_size` of 1MiB means a GET is a long series of small block fetches. Raising it to **10MiB** (and setting `compression_level: none` - image layers are already compressed, zstd on top is wasted CPU) took single-stream from **111 to ~260 MB/s**. We tested 16MiB too: no further gain, so 10MiB it stayed.

This turned out to be the last single-stream improvement of the whole hunt - nothing after this point changed that number.

## Fix #3: the hardware crypto that wasn't there

During another single-stream test, one HAProxy pod used a full CPU core. For one TLS stream? And there was that ChaCha20 cipher again. So: `openssl speed` on the storage nodes.

| Node | AES-128-GCM, per core |
|---|---:|
| the bare-metal node (Xeon E5-2660 v3) | ~2.24 GB/s |
| the Proxmox VMs ("QEMU Virtual CPU version 2.5+") | **~406 MB/s** |

~406 MB/s is software-AES speed. The VMs' `/proc/cpuinfo` *advertised* the `aes` flag, but the generic QEMU CPU model doesn't actually pass AES-NI through - OpenSSL quietly fell back to software, and that's also why TLS kept picking ChaCha20: on that hardware it genuinely *was* the faster cipher. So check with `openssl speed`; don't trust the flag.

The fix was one setting on the infra side - Proxmox VM CPU type -> `host` (plus a power-cycle) - and crypto jumped to 2.9-4.4 GB/s per core on every VM.

And here's the honest part: **single-stream pulls didn't improve.** Garage was serving one GET at ~180-260 MB/s - *below* even the broken 406 MB/s software-crypto ceiling, so crypto was never the single-stream limit. What the fix bought was aggregate headroom - many concurrent TLS streams no longer competing for software-AES CPU - and faster TLS for everything else running on those VMs.

## Fix #4: the overlay network

Next suspect: the network itself. We run k3s with its embedded flannel in VXLAN mode, and a raw-TCP test between two nodes (no registry, no TLS, nothing but sockets) showed this:

| Path (same node pair) | Single stream |
|---|---:|
| host network (no overlay) | ~1018 MB/s |
| pod network (flannel VXLAN) | ~420 MB/s |

The VXLAN encapsulation runs on the CPU - one flow lands on one core - and on our nodes that cost more than half the line rate (the NIC's maximum byte rate - ~1.2 GB/s on 10GbE). (NIC hardware offload for VXLAN would help, but ours report `tx-udp_tnl-segmentation: off [fixed]` - no hardware support.) One caveat: this penalty is **not a fixed number** - it varies from ~15% to ~2.4x depending on the node pair and the load at the time. If your fleet isn't uniform (ours isn't), test several node combinations, and at different times - low and high load give different answers.

A couple of things improved it:

- **virtio multi-queue.** The infra team raised the VMs' virtio-net queues from **1 to 8**. Concurrent flows now hash to different queues/CPUs: VXLAN *aggregate* went from ~34% to ~75% of host speed. Single-stream: unchanged, by design - one flow still means one queue, one CPU.
- **flannel DirectRouting.** For nodes on the same L2 subnet, flannel can skip VXLAN entirely and route via node IPs. A manual A/B (hand-installed route) measured **418 -> 962 MB/s single-stream** - near line rate. One k3s quirk: there is no `--flannel-backend` value for vxlan+DirectRouting; you have to ship a custom `--flannel-conf` on *every* node. Unlike pure `host-gw`, it degrades gracefully mid-rollout - not-yet-flipped nodes still talk VXLAN.

Also found while chasing why one node's pulls were slow: a batch of worker nodes still had **1GbE NICs** - cold pulls on them capped at ~114 MB/s, which is just 1GbE line rate. Check the client's NIC before blaming the registry.

## The twist: single-stream barely improved

So: blobs bypass the proxies, crypto is hardware-speed, the overlay is at line rate, queues are more than enough. Single-stream pull throughput after all of it: **~180 MB/s under load, ~260 idle**. Essentially the post-`block_size` number from fix #2.

The limit was Garage itself, and it is not a bug. A Garage coordinator serves one GET by fetching the object's blocks (mostly from its replica peers over RPC) with limited per-connection parallelism - round-trips between block operations leave every resource idle. During a sustained single-stream GET, nothing saturates: disk at 0.5-2.7 ms/op and mostly idle (hold on to that disk number - part 3 contradicts it under real load), coordinator CPU at ~55% of its limit, network with ~2x headroom. We checked it wasn't the client either - curl (C) and boto3 (python) land on the same numbers. There is no config option for per-GET read parallelism, and the v2 releases change APIs, not the read path. Garage's own docs describe its throughput as moderate compared to MinIO or Ceph - it's a documented design characteristic, and we'd simply never read it at all :laughing: .

So, why didn't this kill the project? Two reasons, both measured:

1. **Real pulls parallelize.** The container runtime (containerd on most of our nodes, cri-dockerd on some) pulls 3-6 layers concurrently, so a real image pull runs at ~200-250 MB/s, and the fleet-scale aggregate - the thing CI actually exercises - measured **~1.5-1.7 GB/s** across the cluster, scaling with node count.
2. **A cold pull is mostly unpack.** For an 846MB image on a fast node: download ~3s, decompress-and-extract **~21s**. The registry accounts for ~12% of the pull time; the rest is node-side CPU and disk. Doubling the download speed would change a 24s pull insignificantly.

**What actually improves pull time is image size, not registry speed.** And most pulls in a healthy CI are layer cache hits that never touch the registry at all.

{{< admonition type=tip >}}
Trust no single counter. Every aggregate number above was cross-checked three independent ways over the same window: client-side downloaded bytes, Garage's own `block_bytes_read` counter, and HAProxy's frontend egress counter - they matched within a few percent. Our metrics platform's per-2-minute "peak" meanwhile showed a third of the real rate - a ~100s burst averaged down inside wide buckets. Integrate totals over the window; don't trust short-bucket rates.
{{< /admonition >}}

## Was it worth it? Old vs new

Throughput was on part 1's goals list because of what slow pulls do to CI jobs: they hit the 1-hour timeout and die. So beyond peak numbers, the comparison that matters is consistency - the same 4.4GB image (the main CI test image), same quiet conditions, pulled from both registries at the runtime's default pull parallelism:

| | New (Harbor + Garage) | Old registry |
|---|---:|---:|
| Pull, 3 parallel connections | **206 MB/s, every run** | 31 MB/s, one round stalled for 90s |
| Pull, 6 parallel connections | 224 MB/s | 275 MB/s (when healthy) |

The old registry could be *faster* on a good day - and would intermittently stall hard even on a quiet system. The new one trades a slightly lower peak for **consistency**: in an early load test of ~980 concurrent CI jobs, registry pull failures were zero. That consistency, not the peak number, is what killed the slow-pull job timeouts from part 1.

## What I took away

1. **Measure per layer, removing one variable at a time.** Direct vs +LB vs +proxy told us in one afternoon what weeks of whole-system tuning would not have.
2. **Separate the data plane from the control plane.** A registry should answer "where are the bytes", not serve them. The 302-redirect-to-S3 pattern removed two proxy hops and was worth 8.5x per stream.
3. **Verify hardware acceleration actually works.** A CPU flag in a VM is a claim, not a fact - `openssl speed` confirms it, and the negotiated TLS cipher is a cheap tell.
4. **Overlay networks cost per-flow CPU**, and the cost varies by node pair and load - measure before declaring a ceiling. Multi-queue lifts aggregate throughput only; DirectRouting raised the single-flow network ceiling too (2.3x for us) - it just could not lift a registry stream that was never network-bound.
5. **Know whether your workload is single-stream or aggregate.** We kept fixing the network's single-flow ceiling while Garage's per-GET serving sat below every one of them. The fixes still mattered - for the fleet, not for one stream.
6. **Pull time is dominated by image size and unpack**, not registry throughput. The best optimization available was never in the registry: fewer and smaller image layers, and better layer caching.

With throughput sorted and dashboards green, we started flipping real load onto the new registry. Part 3 is what happened next: the storage layer fighting back - full disks, a GC storm with a million tombstones per node, [the fstrim weekend]({{< ref "/posts/reclaim-deleted-disk-space-with-fstrim-in-longhorn" >}}), and the bare-metal migration that finally fixed the layer we'd never looked at.
