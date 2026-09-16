---
title: "Build resiliency and delivery guarantees in distributed systems"
description: "The story behind persistent-queue: how edge telemetry collection led to a durable, at-least-once queue, and how a tokio channel plus a KV store played a role in that"
date: 2026-09-16T09:00:00+09:00
lastmod: 2026-09-16T09:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["rust", "distributed-systems", "queues", "mpsc"]
categories: ["Rust"]
series: []

toc:
  enable: false
---

I've been spending a bit more time lately - whatever little spare time I have outside work and family - on a few open source projects. One of them is [persistent-queue](https://github.com/rolandjitsu/persistent-queue): a durable, at-least-once, MPSC queue.

I wanted to take some time to explain the story behind it.

For the past few years I've been working on very similar systems, though in different industries: telemetry collection and ingestion from edge devices. And every single time, the same scenarios came up:

1. Patchy networks with high packet loss and/or low-bandwidth connections.
2. Recurring network outages.
3. Occasional power outages.

When you're dealing with telemetry, there's usually data you can live without and data you absolutely need to get your hands on - it's critical to the business in some way.

So when you build your own telemetry collection - because rolling out an out-of-the-box solution isn't feasible for some reason - you need to build in some resiliency for the scenarios above.

There are many ways to go about it. To survive the patchy-network / low-bandwidth scenario, you can probably get away with something like this:

```rust
let (tx, mut rx) = mpsc::channel(32);

// read from sources, hand each item off over the channel
tokio::spawn(async move {
    loop {
        let data = read_from_source().await;
        // this blocks when the channel is full - backpressure at the source
        if tx.send(data).await.is_err() {
            break;
        }
    }
});

// consume from the channel and ship it out over TCP
tokio::spawn(async move {
    loop {
        tokio::select! {
            Some(data) = rx.recv() => {
                send_with_retry(data).await;
            }
            _ = token.cancelled() => break,
        }
    }
});
```

The above gives you some buffering - so you can keep reading data while a send is failing - and backpressure at the source: when the channel is full, the reading stops. Add a retry in the send path and you also get a guarantee that every data point the channel receives eventually gets sent, even if a network failure happens every now and then. It isn't perfect, but it works fine as long as you accept some limitations:

1. Backpressure at the source means data collection stops when the buffer is full.
2. Your buffer will eventually get full, so sizing it becomes a "find the right balance" game.
3. It's all in memory - a power outage wipes whatever is in your buffer 😬.

Now let's say I want to survive the power-outage scenario too, at least at the buffer level. What we need is to persist the data that goes through the channel somewhere. One way to do that:

```rust
let (tx, mut rx) = mpsc::channel(32);

// on startup, reload anything that was buffered but never sent
for (id, data) in store.iter() {
    tx.send((id, data)).await.ok();
}

// persist first, then hand off - the key is a correlation id
tokio::spawn(async move {
    loop {
        let data = read_from_source().await;
        let id = new_correlation_id();
        store.insert(id, &data); // the durable buffer
        tx.send((id, data)).await.ok();
    }
});

// ship it, then drop it from the store once it's out
tokio::spawn(async move {
    loop {
        tokio::select! {
            Some((id, data)) = rx.recv() => {
                send_with_retry(data).await;
                store.remove(id); // only after a successful send
            }
            _ = token.cancelled() => break,
        }
    }
});
```

This suddenly becomes more complex to work with. But it gives you the guarantee that what's in the buffer is never lost (unless, of course, the program shuts down mid disk write), and you still keep the backpressure and buffering the channel gives you. It still has limitations:

1. The buffer will get full, eventually (just like before).
2. If the data you store is variable-size, your memory footprint may blow up.

Still, you've now built some resiliency into your system, and it should get you further. Switch to a weight-based channel like [weighted-mpsc](https://github.com/rolandjitsu/weighted-mpsc) and count bytes, and you also get a guarantee that your program won't eat away at the memory reserved for other critical systems.

That's great 🎉. But now you need the same functionality in other places. At this point you start abstracting pieces of the code away to cut down on boilerplate and duplication.

That's how I ended up building [persistent-queue](https://github.com/rolandjitsu/persistent-queue). I wanted the same MPSC semantics I get from a channel, but with durability - so I can guarantee at-least-once delivery - and byte-aware buffering - so telemetry collection doesn't steal memory from the critical systems.

I should be upfront that the idea isn't new to me: an earlier version of it was something I designed and built together with [Emma Lee](https://www.linkedin.com/in/yuehjung-lee-1835a822b/), who did most of the initial implementation while I drove the design and the review feedback. persistent-queue is a clean reimplementation, written from scratch - significantly different code, same core idea.

With it, the above example turns into:

```rust
// one durable queue instead of a channel + a kv store + the reload dance
let (tx, rx) = Builder::new(SledStore::open("/var/lib/app/queue")?)
    .max_bytes(64 << 20) // byte-aware backpressure, not a count
    .open_async()
    .await?;

// just push - it's on disk before push returns, and it blocks on the byte budget
tokio::spawn(async move {
    loop {
        let data = read_from_source().await;
        tx.push(data).await.ok();
    }
});

// reserve, ship, ack - a crash before the ack redelivers the item on restart
tokio::spawn(async move {
    while let Some(item) = rx.reserve().await? {
        send_with_retry(&item).await;
        item.ack().await?;
    }
});
```

No correlation ids, no remove-on-success, no startup reload - `reserve`/`ack` is the whole contract 😌. An item isn't gone until you ack it, so anything in flight when the power drops comes back when you start up again.

But then I got creative and started questioning my initial impl. I was using sled at first, and I thought: sled is fine, but what if I want more throughput? I'd heard of this newer KV store written in Rust that looked pretty good - [redb](https://github.com/cberner/redb) - so I gave it a try. After throwing some benchmarks together, the numbers didn't show a significant improvement. Then I'd heard good things about [RocksDB](https://rocksdb.org) - I know, it's not native Rust, but the tradeoffs are worth it if it proves to be better. And it was: at least 4x faster.

I didn't want to force users onto one backend, so I added support for all of them behind feature flags. Swapping between them is a one-line change:

```rust
// the only line that changes to switch backends:
let store = RocksStore::open("/var/lib/app/queue")?; // was SledStore::open(...)
```

I went further - because, you know, having tools that can write code for you opens the gate to doing unnecessary things 😂 I figured maybe grouping the sync of messages to disk would make things faster. It did, but only slightly. Behind a feature flag it went, too.

Why stop there?! What if users want typed data in and out? Sure - let's add codec support, also behind a feature flag. At first it was just serde + bincode. But I wanted to see if a zero-copy approach could do better, and found [rkyv](https://rkyv.org). Sure enough, the rkyv path was an order of magnitude faster when I benchmarked it. Still, let users choose - so each codec went behind its own feature flag. I won't paste the API here - the typed and zero-copy examples are in the [README](https://github.com/rolandjitsu/persistent-queue).

I kind of got tired at this point 🫩, and stopped after adding an async facade via tokio to make it easy to drop into tokio-based projects.

In the end, I was happy with the outcome.

But back to the title - distributed systems. I talked about edge devices, so how does this relate? Distributed systems aren't all that different from "distributed" edge devices. Your services can hit the same scenarios I described at the beginning, especially when they're spread across regions. At that point you may need the same resiliency and delivery guarantees, and you're likely to build something along the lines of what I ended up with.

P.S. I do recommend using battle-tested telemetry agents like [Vector](https://vector.dev) when you can. Rolling your own comes with a maintenance cost you - or your team - may find too expensive to pay later. But if you don't have that option and have to roll your own, give [persistent-queue](https://github.com/rolandjitsu/persistent-queue) a try.
