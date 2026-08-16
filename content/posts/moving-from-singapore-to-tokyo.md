---
title: "Moving from Singapore to Tokyo"
description: "Why I left Singapore after five years, what the move to Tokyo actually took, and what I'm building now."
date: 2026-08-09T15:20:00+09:00
lastmod: 2026-08-09T15:20:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["thoughts"]
categories: []
series: []

toc:
  enable: false
---

It's been a while. A long while :sweat_smile: .

The last time I wrote here I opened with almost the same line - back then the excuse was becoming a father. This time it's ~~bigger~~ (well, nothing's really bigger than becoming a father) this: I packed up my life in Singapore and moved to Tokyo. That was October 2024, so this update is running about two years late :grimacing: .

So here's the catch-up: why I left, what the move actually took (spoiler: nothing like what you see on YouTube), and what I'm building now.

## Leaving Singapore

I spent five+ years in Singapore, and they were good years. But somewhere around year five I realised I'd gotten *too* comfortable. Life was easy, predictable, and small - the same routine, the same places, the same tropical heat every single day of the year. No countryside to escape to, no seasons, no real sense of *going somewhere new*. Comfort is a trap, and I was deep in it.

Work played a part too. I spent five good years at [Transcelestial](https://transcelestial.com/), but by the time I left, the company was still finding its focus and I was ready for a new chapter. Five years felt like enough.

So I started looking - not just for a job, but for a different life.

## Why Tokyo

I searched across a few regions: Tokyo (safe, family oriented, lots to see), Singapore (stay, but different), Australia (been there once and loved it), and Switzerland (spent some years there and I liked it + it's closer to home). Tokyo was the one that panned out.

On paper it ticked every box: a safe place to raise kids, with more of a family-first culture (or so it looked from afar), loads of green, four actual seasons, and an endless list of things to see and do. After five years of sameness, that was exactly what I was looking for.

## The move (aka the YouTube dream)

Here's the part nobody - well, not in the posts I watched - puts on YouTube (and other sources).

The visa alone took about 2.5 months and a trip back home to sort out at the Japanese embassy. Then the shipping: we hired a company to move our things, and it *went under and kept our money without shipping a single box* :rage: . We ended up throwing a bunch of stuff away and having a friend (thanks Kelvin!) post the rest - our kid's toys travelled to Japan by mail :package: - he was happy.

Then you land, and the real fun begins:

1. The paperwork is all on *paper*. Nothing online. You do it in person, at the ward office, in Japanese - which we still don't speak :sweat_smile: .
2. Finding a school is hard. We went with a local pre-school, but the enrolment process is intense and we ended up waiting until *April* to actually start (that's 4+ months w/o school).
3. We landed in a serviced apartment first - tiny, expensive, and cramped for two adults and a kid.
4. Finding a place to rent was its own saga. And it ain't cheap either!

All in, the move cost us a lot more than the relocation package covered. Not the glossy Tokyo relocation story you tend to see online.

## Tokyo, honestly

I'll be blunt: Tokyo isn't the paradise the internet makes it out to be. Almost two years in, here's my honest take.

The good is genuinely good. It's *safe*. The transport is unreal (I thought Singapore was great) - you can get anywhere, on time, every time. Rent is actually lower than Singapore. Healthcare covers around 70% of costs.

The hard is real too. Socially it's been tough - I haven't made friends outside of work, which is a big change from Singapore where we'd built a proper circle of families with kids the same age as ours. Healthy and/or western food is expensive (fruit especially). And navigating Japanese doctors when you don't speak the language is... an adventure :grimacing: .

Would I do it again? Yeah. But I walked in with postcard expectations, and reality charged me the difference.

## The new job

The move was for a role at [Mujin](https://mujin.co.jp/).

Mujin does warehouse automation. We don't really build the robots themselves - beyond grippers and some custom hardware - we build the *software* that controls them: robots, conveyors, lifts, AGVs, the lot. It's a different world from laser comms, but not as far as you'd think - it's still hardware-adjacent systems software, which is exactly where I like to be.

I joined to lead cloud engineering: build a platform for fleet management and observability, and grow a cloud team - not too far from what I'd been doing before. Since then the role has grown a lot. I picked up DevOps and the Release team, and today I'm lead platform engineer running our engineering enablement team. That's been a journey :sweat_smile: .

Mujin still runs a lot like a startup, with just enough process layered on top. A big part of my job is nudging the software toward being more cloud-native - both cloud and on-prem - and levelling up our engineering practices along the way. It's greenfield, it's messy in the way early platforms always are, and it's genuinely interesting.

## ...and a new stack

The other big shift: I've gone almost all-in on [Rust](https://www.rust-lang.org/).

Everything my team builds - services, CLIs - is Rust. (The rest of the company lives in different stack.) A lot of that choice came down to a hard constraint: some of the software we built on my team often had to run on the Mujin controllers, where CPU and memory are reserved for critical systems, and Rust's low footprint made it an easy call.

Day to day I spend a lot of time on CI and observability - making builds and tests faster and able to scale, including running CI in the cloud elastically so we can burst from zero to whatever we need and back down. And I've been building and consolidating a set of services into a proper platform: observability, remote access, log collection, and more.

I actually started dabbling with Rust in my last few months at Transcelestial, but Mujin is where I took it seriously and built everything with it. I've got a lot to say about the Go -> Rust switch - what I love, what drives me up the wall, and how I actually use it - so that's the next post.

## So, the blog

Anyway - that's where I've been. New country, new company, new industry, new stack.

The plan is to write more again, and you'll probably notice the topics drifting toward what I'm doing now: robotics and platform work, a lot of Rust, cloud and on-prem infra, and how AI has changed the way I build (another post I owe you).

And if you're on the comfortable-but-safe path and it's starting to feel a little *too* comfortable - maybe that's your sign :wink: .
