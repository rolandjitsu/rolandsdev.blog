---
title: "The registry storage war: full disks, a GC storm, and bare metal"
description: "Part 3 of the OCI registry series: a neighbor fills a disk and corrupts a database, a million deletion tombstones per node slow pushes down to 47 minutes, and the fix is three scavenged bare-metal nodes."
date: 2026-11-08T16:00:00+09:00
lastmod: 2026-11-08T16:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["kubernetes", "garage", "harbor", "longhorn", "storage"]
categories: ["DevOps"]
series: ["an oci registry for ci"]

toc:
  enable: true
---

[Part 1]({{< ref "/posts/replacing-our-ci-registry-with-harbor-on-garage" >}}) was the design and the rollout; [part 2]({{< ref "/posts/hunting-for-registry-throughput-layer-by-layer" >}}) was the throughput hunt. By early October the new registry was fast, the dashboards were green, and we started flipping real CI load onto it.

This part is the ten days that followed. Remember the storage stack diagram from part 1, with Garage sitting on Longhorn, on a VM virtual disk, on Proxmox's network storage, on physical disks somewhere in a rack? This is the part where that stack screwed us 😅

## A neighbor fills a disk

Midweek, one Garage node went into an error storm. The disk it stored data on had hit **100% full** - and not because of the registry. The Garage volume shared its disk with a build-cache service whose volume had grown about 50% *past* its PV size: a default weekly Longhorn snapshot was pinning old blocks, and snapshots live outside the volume's nominal allocation. The neighbor's overflow filled the registry's disk.

It gets worse. When a Longhorn disk fills up completely, writes don't fail with `ENOSPC` - the volume layer returns **`EIO`**, a generic I/O error. The application never gets told "disk full"; it just sees hours of failing writes. Garage's metadata store on that node is an [LMDB](https://garagehq.deuxfleurs.fr/documentation/reference-manual/configuration/#db-engine-since-v0-8-0) database, and hours of `EIO` mid-write left it corrupted: `MDB_CORRUPTED: Located page was wrong type`.

The recovery was manual surgery: stop the node, delete the corrupted metadata DB while keeping the node's identity key, and let it rejoin and rebuild its metadata from the other replicas - the resync queue started at ~72k items and drained overnight. RF=3 doing its job, again.

The structural fix: the Garage disks became Garage-only, and the neighbor moved elsewhere. **A shared disk makes every workload on it part of your failure domain.**

## Two days of firefighting 🔥

Meanwhile the actual capacity problem was compounding. CI inflow was higher than planned for, and by Thursday the five nodes were down to **17-21% free**. Every emergency measure made things worse in its own way:

- An aggressive overnight cleanup removed images that pipelines were actively using - jobs failed on pulls of images that had existed when the pipeline started. Thousands of jobs were hit in a single day.
- Push errors exposed that Harbor's database connection limit (100) was saturated; raised to 300.
- Most S3 traffic turned out to be landing on a single Garage node; rebalanced across all of them.
- The real fix - growing the disks, which requires a VM reboot - was attempted one node at a time with the registry live, and had to be rolled back: some blobs were reachable only through the node we took down (an under-replicated tail from an earlier 3 -> 5 node expansion), and CI started failing with `blob unknown`. An hour and a half later the node was back and we had gained zero bytes.

It was Friday afternoon, the expansion was deferred, and a tightened retention policy plus a garbage collection (GC) run were supposed to free enough space to get us through the weekend.

## The weekend of the full disks

The weekend did not go as planned - but not for the reason I expected. Saturday morning two nodes were at 95%+ while Garage itself reported only ~40% used: the GC had worked, Garage had deleted the data, but the disks never cleared it. That story has [its own post]({{< ref "/posts/reclaim-deleted-disk-space-with-fstrim-in-longhorn" >}}): deleting files doesn't free blocks on thin-provisioned storage until something runs `fstrim`. A manual trim reclaimed ~3.9TB across the fleet within minutes, and a scheduled trim job made it permanent. Crisis deferred, weekend saved.

## Maintenance Monday

Monday was a holiday, so naturally we scheduled everything at once 😅: the long-deferred disk expansion (four of the five volumes grew by roughly a terabyte each), the fleet-wide flannel DirectRouting rollout from part 2, runner upgrades, and node memory doubling. The day itself went fine.

By evening, the median push had slowed to **47 minutes**, cluster-wide. The registry was serving; it was just unusably slow. Not what you want at the end of a maintenance day that was supposed to *fix* capacity 😑.

## What made the pushes slow

The usual suspects, each ruled out by measurement: not the network (a node-to-node test moved 826 MB/s with 5ms round-trip latency), not the just-grown volumes (disk latency moved with load, not with volume size, and all volumes reported healthy), not data loss (zero errored blocks). Tuning Garage's resync concurrency up made it *worse* - the most aggressive setting drained 8x slower than the gentle one.

The actual cause: all the mass deletions of the preceding days - retention runs, emergency cleanups, the GC - had created **~1 million deletion tombstones per storage node**, and Garage defers tombstone cleanup by ~24h, so they all triggered together - on the same day the layout changed. The internal logic that handles them is unthrottled: it saturated CPU (30-40 cores per node), starved the inter-node RPC until nodes couldn't answer each other's pings, and monopolized the metadata DB's single-writer lock. Live pushes share that RPC and metadata path, so they were collateral damage. And while the cluster barely kept up, Harbor's **hourly** GC kept feeding thousands of fresh deletions into it. Hourly was my own doing - I had assumed frequent small cleanups would put less load on the cluster than one big batch every day or two. Usually true; mid-storm, it only grew the backlog.

The fix was mostly restraint:

- **Stop the feed**: GC rescheduled from hourly to once a day, off-peak. With the trim job and ~65% free disk, hourly reclamation had no purpose anyway.
- **Let it digest**: with inflow stopped, the tombstone backlog drained in ~4.5 hours.
- **Pin the pacing**: resync concurrency fixed at moderate via GitOps so a pod restart can't reset it; Garage memory 32 -> 64GB to keep the hot set cached.

| metric | during the storm | after |
|---|---:|---:|
| push (main image, median) | 2841s (47 min) | 194s |
| internal RPC ping failures | 25-37 per 2.5 min | 0 |
| storage-node CPU | 30-40 cores | <0.5 core |
| cleanup backlog | ~1M items/node | 0 |

The lesson I learned: **treat deletes like writes**. A mass deletion is a batch workload with real I/O and coordination costs, deferred to a time you don't choose. Pace it like you'd pace any batch job, and don't stack it on top of layout changes and node reboots in one day.

## The root cause was the disks

Rescheduling GC fixed the trigger, not the root cause. The reason a tombstone backlog *could* bring down the whole cluster - and the common thread under the corruption, the slow pushes, and the firefighting - was what the storage sat on. Measured over a day of production load:

| | the 4 VM nodes | the 1 bare-metal node (still on Longhorn) |
|---|---:|---:|
| disk read latency, p99 | **15-52 ms** | 3.4 ms |
| disk utilization for the same work | 80-99% | ~3% |

Same cluster, same workload, 5-15x latency difference. The VMs' disks went through Longhorn, a virtio virtual disk, and Proxmox's network storage; the bare-metal node also ran Longhorn, but with nothing underneath it - its volume sat on a local disk, no virtio, no network storage. Every incident so far had this amplifier underneath it. None of it was an unreasonable setup - Longhorn is the standard way we provision storage, and the VMs existed because there was no spare bare metal when we started - but it turned out to be the wrong stack for a store that replicates itself and works the disks this hard.

We couldn't buy hardware overnight and there was none to spare, so we scavenged: three bare-metal nodes pulled out of a CI test pool, their NVMe drives given to Garage directly - a plain local PV on the host disk, **no Longhorn, no network storage**. Durability is Garage's RF=3; a dead node loses one replica and rebuilds from peers. The swaps ran live, one node at a time: drain the node's data out, repoint that Garage member to the new host (PV names matching the StatefulSet's numbering so each pod finds its volume), let it refill. During the migration we also rolled part of CI back to the old registry on purpose - no reason to risk more job failures through the churn.

## The draining node that kept serving traffic

The second swap's overnight drain produced the last real incident, and it's a subtle one. Between 23:12 and 07:19 JST, ~250 CI jobs failed on pulls - 500s from Harbor, 403s on direct blob downloads - while CPU and disks sat near idle. This was not load.

Removing a node from Garage's layout starts the drain, but the node's process must keep running (it's the data source) - and to the S3 load balancer it still looked perfectly healthy, so it kept receiving client traffic. What I didn't know: a node removed from the layout drops out of internal metadata replication, so it eventually stopped recognizing the registry's S3 access key - and answered **every request with `403: No such key`**, which Harbor surfaces as a 500. One poisoned backend out of six, with connection keep-alive pinning clients to it in bursts.

Harbor's logs matched the finding exactly: the error count starts at the minute the layout change was applied and stops at the minute the drained node's pod was deleted. Nothing was lost - every affected image served fine the next morning.

Two fixes came out of that night. The runbook we put together for these swaps now drains nodes **as gateways** - they keep serving S3 correctly while their data moves out - instead of removing them from the layout. And I finally set up **alerting**: until that week we had dashboards but not a single alert, and every incident in this series was discovered by a person. Thirteen alert rules now page us - storage fill, lost write quorum, 5xx rates, backend down, degraded volumes - and the "No such key" issue is documented. **Dashboards are for debugging; alerts are for finding out.**

## What bare metal bought

With the three swapped nodes on local NVMe and the Harbor stack moved off the VMs entirely, the first churn-free day looked like this:

| | VM era | after the migration |
|---|---:|---:|
| image-pull failures, share of all CI jobs | 4.63% | **0.22%** |
| pulls over 10 min (the job-killers) | 0.28% | **0 of ~79,000** |
| pull duration p90 / p99 | 89s / 288s | 29s / 122s |
| registry serving 5xx (all layers) | constant background | **zero, under full load** |
| one client, 8 parallel streams | - | **~1.2 GB/s** (its own 10GbE line rate) |

One honest caveat: the same week, our build images became reproducible. Previously, every build produced new layer digests even when nothing changed; now identical layers keep identical digests and dedup on push. Typical pushes dropped to ~82s, and pushes over 10 minutes went from ~30% to ~6%. Both changes landed together, so I can't fully separate their contributions; the long-tail numbers above are the combined effect.

The slow-pull timeouts that started this whole series - 12.2% of pulls over 5 minutes on the old registry - are at 0.02-1.4% depending on the window, and zero over 10 minutes. More important than the numbers is how failures happen now: the old registry failed jobs *slowly*, burning an hour before dying; now things fail fast and page us.

## Where things stand

The cluster is a mix right now - most of it on bare metal (the newest nodes on local NVMe, one still on Longhorn), some VMs still in the mix - and the plan is to finish the move and retire the VMs. The noisy-neighbor lesson also got enforced for real: the storage nodes now run only registry and storage workloads, so no unrelated volume can fill a Garage disk again. Moving everything else off them caused a few short blips, but nothing major. The more important change is the position we're in compared to two weeks ago: effective capacity sits at roughly twice the post-GC peak, the trim job keeps deleted space actually free, GC runs paced and off-peak, and the whole path is covered by alerts.

## What I took away

1. **A shared disk is a shared failure domain.** The registry's worst early incident was caused by a workload that had nothing to do with the registry. Stateful stores should get their own disks.
2. **A full Longhorn disk fails with `EIO`, not `ENOSPC`** - applications see generic I/O errors and some (like LMDB) corrupt. Treat storage capacity alerts as critical; by the time writes fail, it's surgery time.
3. **Treat deletes like writes.** In Garage, every deletion leaves a tombstone that has to be processed later - and other replicated stores have similar deferred cleanup. Mass deletion creates load that arrives on its own schedule; pace it, and never stack it on top of layout changes and reboots.
4. **Stop the thing causing the failure first, then dig for the root cause.** Rescheduling GC ended the incident; the 15-52ms virtualized disks were the root cause, and fixing them was a separate project.
5. **Count the layers between your store and the disk.** Each one costs latency (ours: Longhorn -> virtio -> network storage), and a store that already replicates itself does not need replicated storage below it. Keep the stack as short as you can.
6. **A draining node can look healthy to your load balancer** while failing every request. Drain through a gateway state, and write down the exact error that tells you this is happening (ours was `No such key`).
7. **Dashboards are for debugging; alerts are for finding out.** Every incident in this series was found by a human until we wrote the alert rules. Write them before the rollout, not after week five.

That's the war. The registry has been quiet since - fast pulls, zero serving errors under full CI load, and incidents that page us instead of surprising us. If you run a store that replicates itself, go look at what its disks actually sit on. We should have.
