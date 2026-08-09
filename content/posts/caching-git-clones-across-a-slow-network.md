---
title: "Caching git clones across a slow network"
description: "A small read-only proxy that serves CI git clones from a local mirror, so only the delta ever crosses the WAN."
date: 2026-08-09T10:00:00+08:00
lastmod: 2026-08-09T10:00:00+08:00
draft: false
authors: ["rolandjitsu"]

tags: ["git", "ci", "rust"]
categories: ["DevOps"]
series: []

toc:
  enable: false
---

If you run CI on cloud runners while your git server lives somewhere else - on-prem, another region, the far end of a slow VPN - you have probably watched a lot of time (and egress) disappear into cloning the same repositories over and over.

The setup I kept running into: an on-prem GitLab, an elastic fleet of runners autoscaling in the cloud, and a handful of repos in the tens of gigabytes. Multiply that by hundreds or thousands of jobs a day and every single one does a fresh clone across the WAN. The link is the bottleneck, the egress is not free, and the runners end up spending more time fetching code than running it.

Shallow and partial clones (`--depth`, `--filter=blob:none`) help and you should use them, but plenty of jobs still want full history, and you are still paying per job and per runner. What I actually wanted was simple: put a cache next to the runners so each repo is served locally, and only what is genuinely new crosses the WAN.

## Why not just cache it?

The obvious answers do not quite work.

A generic HTTP cache (nginx, Varnish, etc.) does nothing here. A `git fetch` over smart HTTP is a negotiation: the client advertises what it already has, and the server computes a packfile on the fly for that exact request (`POST /git-upload-pack`) - see [the Git HTTP-based protocols](https://git-scm.com/docs/gitprotocol-http#_smart_clients). There is no static response sitting there to cache.

{{< admonition type=note >}}
This is the part that surprised me the first time. `git clone` looks like an HTTP download, but the interesting response is generated per request from what the client says it needs. A byte cache in front of it is useless.
{{< /admonition >}}

A scheduled full mirror (GitLab pull-mirror, Gitea) works, but it is the opposite of lazy. You enumerate and replicate every repo on a timer, it can lag the commit CI pushed thirty seconds ago, and you now keep a full copy of everything wherever the mirror runs - which is exactly what you may not be allowed to do when the origin is sensitive.

## A small read-only proxy

So I built [git-cache-proxy](https://github.com/rolandjitsu/git-cache-proxy) - to be upfront about how this got made: I designed it and drove the work, but Claude wrote most of the actual code. It's a read-only caching proxy that sits between the runners and the origin and does the least it can get away with:

- On the first request for a repo it does a `clone --mirror` into a local bare repo.
- On every request it runs an incremental `git fetch` from upstream *first*, then serves from the mirror. So a client always gets the ref it asked for - including one pushed a second ago - but only the new objects cross the WAN.
- Concurrent clients for the same repo are coalesced into a single upstream fetch, and a short TTL collapses bursts. When fifty runners wake up at once, upstream sees one fetch.
- It is pull-only. It never pushes back to the origin and never writes upstream. This was the part our security team cared about: we couldn't just set up another GitLab or Gitea in the cloud and mirror sensitive repos out to it, when everything sits behind a firewall on-prem for a reason. We needed something with a small blast radius - a cache that only ever pulls what a job asks for, and never becomes a second, full copy of everything living outside the perimeter.

All the wire-protocol work is delegated to the system `git` binary, so protocol correctness - v2, shallow, partial clones - comes for free. Reimplementing pack negotiation was not in scope or an option.

Point a client at it with git's `insteadOf`:

```sh
git config url."http://proxy:8080/".insteadOf           "https://git.example.com/"
git config url."https://git.example.com/".pushInsteadOf "https://git.example.com/"
```

The first rewrite routes clones and fetches through the proxy; the `pushInsteadOf` sends pushes straight back to the origin, because the proxy will not take them.

Run it as a container (it bundles `git`) or install the binary:

```sh
docker run --rm -p 8080:8080 ghcr.io/rolandjitsu/git-cache-proxy --upstream https://git.example.com
# or
cargo install git-cache-proxy
```

## Does it help?

Numbers from the repo's own benchmark - a 64 MB repo over an emulated 20 Mbit/s, 60 ms link:

| Scenario                      | Clone time | WAN bytes |
| ----------------------------- | ---------: | --------: |
| Direct clone (today)          |     28.1 s |     64 MB |
| Via proxy, cold (runner 1)    |     28.5 s |     64 MB |
| Via proxy, warm (runner 2..N) |      0.6 s |     ~0 MB |

The first runner pays the usual price. Every runner after it clones in well under a second and pulls nothing across the WAN until there is an actual new commit to fetch. The saving scales with the size of your fleet and the cost of your link - which, if you got this far, is probably the whole problem.

## Things to note

- There's no TLS built in - it serves plain HTTP. If you set a serve token, it travels in the clear, so put it behind something that terminates TLS on any network you don't fully trust.
- There's no per-repo auth. Reaching the port means reading every mirrored repo, so treat "can reach the proxy" as "can read everything the proxy can read" and lock the network down to match.
- The on-disk cache grows unbounded for now (no eviction yet), so give it its own volume and keep an eye on it.
- It shells out to a real `git`, so the host (or image) needs `git` installed.

The code is at [rolandjitsu/git-cache-proxy](https://github.com/rolandjitsu/git-cache-proxy) (Apache-2.0) and on [crates.io](https://crates.io/crates/git-cache-proxy), with the design notes, the full config, and that benchmark you can run yourself. If you have a similar setup - or a good reason this is a terrible idea - I would like to hear it.
