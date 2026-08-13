---
title: "From Go to Rust"
description: "Five years of Go, then Rust: how I got dragged in, what fought me, what I ended up loving, and who should actually make the jump."
date: 2026-08-13T10:00:00+09:00
lastmod: 2026-08-13T10:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["rust", "go"]
categories: ["Rust"]
series: []

toc:
  enable: true
---

I'd been writing [Go](https://go.dev/) for 5+ years. I didn't go looking for [Rust](https://www.rust-lang.org/) - it got dragged into my world, and honestly, I was defensive about it at first.

## How I got dragged in

Back at [Transcelestial](https://transcelestial.com/), a colleague of mine, was building a tool to flash Raspberry Pis using an A/B partition setup (partition A is active, B is passive; a failed boot falls back to the partition that still works). He wrote it in Rust.

My first reaction was skepticism, maybe a bit of gatekeeping: *why are we bringing another language into the mix?!* Our team's world was Go. But the tool was small, self-contained, and - well, it had already happened :sweat_smile: - so it was a decent, low-risk way to see what Rust was about.

Then came another CLI, this time a wrapper around a bunch of queries to one of our backends. I remember staring at the code thinking *wtf is all this*: `fn` instead of `func`, `impl`, `Option`, `Result`, `#[derive(...)]`. It was confusing and hard to digest.

But the more I read about Rust in the days that followed, the more it started to make sense - and the more interesting it got.

## The project that made it real

Then an opportunity landed on my plate: rebuild, from the ground up, one of the CLIs we used to operate our the devices, plus a set of services running on the device to stream its state (health, sw/fw info, sensor data, that kind of thing).

Two things pushed me toward Rust for this.

**Messaging.** There was a lot of mpsc- and mpmc-style communication between the services, and between the services and the CLI. In Go, mpsc is trivial - that's just a `chan`. But mpmc, where *every* consumer needs to receive *every* message a producer sends, is more of a pain. Rust had a crate that gave me exactly that.

**Memory.** The services on the device needed a low, bounded memory footprint, and I wanted some control over it. In Go, the GC manages memory for you, with very little say over how and when. In Rust, memory is yours to think about, like in C or C++ - nobody cleans up after you. (Well, Rust gives you a pile of compile-time guarantees so you can't do anything *too* stupid, but you can still absolutely mess up at runtime.)

So I gave it a shot and built both the CLI and the services in Rust.

One of the first things I had to figure out was generating code from our `.proto` files, the way we did in Go (generated code checked in). [prost](https://github.com/tokio-rs/prost) came to the rescue, and implementing the service logic on top of the generated code was surprisingly smooth. The CLI was the same story - [clap](https://docs.rs/clap/) had it covered. The one real surprise was async: I didn't really *get* how it ran at the time, but [tokio](https://tokio.rs/) made it manageable (mostly).

And there I was, writing a pile of Rust - while still maintaining and writing Go. The feelings were... mixed :sweat_smile: .

## Coming from Go

I won't sugarcoat it: the transition wasn't easy.

Some Go concepts carried over (`chan` -> mpsc, struct -> struct, interface -> traits), but a lot didn't. There's a real learning curve - probably smoother if you're coming from C/C++. I started the way a lot of people do: reading [the Rust book](https://doc.rust-lang.org/book/) and the docs it references, poking at things in [the playground](https://play.rust-lang.org/). That felt fine. But reading Rust and *building* with it are two very different things.

The things that hit hardest, roughly in order:

1. **Compiler strictness.** It threw *so* many errors at me. Even reading and understanding them was an adjustment.
2. **No `return`.** You... don't need it?! (the last expression is the value)
3. **Macros.** *What is `#[derive]`?!*
4. **The borrow checker.** This one was - and still is, sometimes - a struggle. It didn't magically get clearer just from writing more code (lots of Stack Overflow and forum reading; AI wasn't a thing back then).
5. **Async.** Just when I thought I had a grip on ownership and lifetimes, along came `async`, `async move`, and futures that need lifetimes, plus moving and copying values and references into async scopes. That cranked the difficulty right back up.

In retrospect, the biggest lesson: **understand the memory model, the borrow checker, and lifetimes early. Do not skim them.** It pays off everywhere.

Was it worth it? Definitely. I regret none of it.

## What I love now

Funny enough, the thing I found most annoying at the start - the strictness of the compiler and the language - is exactly what I love now.

- **No more nil-pointer panics.** I can't count how many I fixed in Go: no matter how many unit and integration tests we had, some dependency would get passed as a pointer and we'd forget to wire it up. In Rust, that's just... not a class of bug anymore.
- **`Option<T>`.** I love this thing. I wish Go had it. It makes optionality explicit - you literally can't use the value without handling the fact that it might be `None`.
- **`Result<T, E>` and `?`.** Completely new to me, but once it clicked it made total sense. No more:

```go
val, err := doThing()
if err != nil {
    return err
}
```

Instead:

```rust
let val = do_thing()?;
```

That `?` alone made me love the language a bit more.

- **`match`.** Just nice.
- **Channels in std.** `mpsc` is right there in the standard library (with `mpmc` on the way).
- **cargo + crates + std.** Go has a big ecosystem, but I find it easier to land on what I need in Rust's std or a mature crate.
- **Tests in the source file.** Makes a lot of sense to me.
- **Generics.** Still a bit on the fence - more robust than Go's, but they get gnarly the moment you mix them with async.

## What still fights me

It's not all sunshine:

- The **`return` oddity** - you *can* use it, you just don't *need* to - can be confusing when the two styles mix.
- **Async + borrow checker + lifetimes** together are still a struggle sometimes.
- **`Box`/`Pin`** are still things I don't reach for often or fully understand.
- **Generics mixed with async.**
- The **ceremony for simple programs.** In Go, a small HTTP server and a few handlers is almost nothing. In Rust there's just a bit more to get going (though maybe that's years of Go muscle memory talking).
- **Build times** can get long with a lot of dependencies - but so can Go's, to be fair.

And what do I miss from Go? Honestly, not much. Rust works well for me, and I'm perfectly happy writing Go too. Depending on what I need, I'll reach for either.

## How I use it today

I can't share too much about my current stack, but broadly: most of what we write runs on [Kubernetes](https://kubernetes.io/) - lots of (micro)services and a few CLIs. Services are mostly gRPC or HTTP, with the usual async ecosystem doing the heavy lifting. We run a monorepo/workspace, which works well.

One thing I'm still chewing on: a lot of code these days gets generated by LLMs, especially by folks new to Rust. I'm genuinely not sure how I feel about it yet - it lowers the barrier, but the foundations still matter, and I don't think you get to skip them. (More on AI and how it's changed my work in another post.)

## The team angle

Bringing Rust into a shop that's heavily invested in their current stack came with some resistance - some of it for perfectly valid reasons. But the folks who *did* write Rust and shipped services to prod seem happy with it: the services are remarkably stable, and you hardly ever have to touch them once they're up.

Onboarding is easier now with LLMs. Hiring is tough - but hiring is tough for any stack (I don't hire for a specific stack anyway; people can upskill).

The part I miss most is **human code review.** LLM review is getting more traction, and it sort of works, but I miss the human side - the debate over what's idiomatic and what isn't, the preferences, the scrutiny. Those discussions tend to produce the best work, precisely because the code is under pressure. An LLM can just be... dismissive.

## Would I do it again? And for whom?

100%, yes.

But it's not for everyone:

- **If Go can do the job, use Go.** :)
- Reach for **Rust** when you have real constraints: memory control, production stability, or needs that live close to the metal.
- If your stack is already Rust, keep going.
- If your stack is Go, don't switch without a compelling reason - like performance beyond what Go can realistically give you.

And if you're making the jump: start with the **foundations**. Read the book and the docs, all of it. Really understand the **borrow checker and lifetimes** (a MUST), and how **async executors and futures** actually work. Then just write code - *you*, not the LLM - and learn from the experience (and the pain).

## The one stat that sells it

In all my time shipping Rust, I've had exactly **two** binaries crash on me. One was the way I handled async retries: recursion that grew the stack until it hit the max and blew up - fixed by switching to an iterative approach. The other was the program trying to read a file that didn't exist.

Two. That's it. Well, there might have been others, but minor enough to forget about it.

Compare that to my time with C++ - and to be fair, I'm no seasoned C++ engineer, so plenty of those were rookie mistakes - where I ran into runtime memory issues constantly.

The journey wasn't easy. But the strictness of the compiler and those runtime guarantees? Worth every bit of the fight.
