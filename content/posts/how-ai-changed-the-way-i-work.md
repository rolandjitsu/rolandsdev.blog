---
title: "How AI Changed the Way I Work"
description: "AI now does most of my writing and a good chunk of my coding. My honest ledger: the wins, the tensions, and the foundations I won't trade away."
date: 2026-08-14T10:00:00+09:00
lastmod: 2026-08-14T10:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["thoughts", "ai"]
categories: []
series: []

toc:
  enable: true
---

A confession to start: a big chunk of the work I used to do by hand is now done by an AI. Not all of it, but a lot - I'd guess 60 to 75% of what my job used to be day-to-day. Including, full disclosure, this very post: an AI interviewed me, I brain-dumped the answers, it did the writing - correcting and reshaping as it went - and I reviewed it and hit publish :sweat_smile: .

I'm mostly happy about that. *Mostly.* Here's my honest ledger of what changed, what I gained, and the parts that quietly worry me.

## The old loop vs. the new one

The change isn't uniform, so let me go job by job.

**Debugging.**
Before: pull the logs and metrics, find where we log that line in the code, dig through the code and the tests, maybe write a test to reproduce or spin up staging. Then Google the meaningful part of the log, read the top 5-20 results (SO, blog posts), dig through the dependency's source, its PRs and issues. Make the fix, deploy, watch it (staging, then prod - sometimes straight to prod).

Now: if the AI has access to the infra or the thing, I ask it to look and summarize; sometimes it reads the responsible code and figures it out, or reproduces it with a test. If it doesn't have access, I paste the log plus links to the relevant code and let it reason. It gets it right maybe 80-90% of the time - depending on how complex and well-documented the system is, and how relevant the logs are. From there the flow is the same as before. (I'm not ready to hand any agent the keys to deploy - more on that later.)

**Writing code.**
Before: for anything big, an RFC first - design, architecture, a feasibility prototype, interfaces, layout, migration - reviewed and approved (impl usually started before approval anyway), then build the thing, test in staging, MR goes draft -> ready, code review, a few rounds of comments, merge, deploy. Small changes skipped straight to an MR.

Now: same flow, different hands. Instead of writing it, I describe what I want and the AI writes the RFC or spec; I review and approve; then it implements the thing the way I would have - and I review it like I'd review anyone's code. Leave feedback, let it fix, and if it fumbles too many times, I take over and finish it myself. Then the usual MR flow to prod. Same for fixes.

One side effect I didn't expect: because it's so cheap to *start* something, I now start way too many things at once, and I keep losing track of what I was doing because I'm context-switching constantly :sweat_smile: .

**Reviewing code.**
Honestly, the *flow* changed the least here - we use AI for a first pass, but that's about it. What changed is the *code* and the *volume*. AI code tends to be more verbose - a lot more comments and filler than a human would write, which gets distracting. And the volume has jumped: I used to review maybe a dozen to 20 MRs a week; now it's 2-3x that (it cools off when people are on leave :)) ). With that much to get through, I'll admit I don't scrutinize each one the way I used to.

**Design, RFCs, docs.**
AI has basically taken this over. The ideas and the direction are mine; the writing and the diagrams are its. Early on I'd go through every sentence to make it sound like me. I've mostly given up on that now - an [AGENTS.md](https://agents.md/) and a few house-style files get it close enough, and when I've got ten things in flight I don't have time to polish. The speed is the win: I can turn thoughts into a decent doc in minutes.

**Admin and the rest.**
Anything I need to compile or write - if it's not sensitive - I hand off and iterate until it's in the shape I want. Case in point: I used to spend a *week* making 10-20 halfway-decent slides. It does that in a few minutes now. *wtf* :exploding_head: .

## What it's genuinely great at

No doubt it helps - with almost everything. Writing code, writing docs, pulling scattered data into something presentable, searching (it doesn't always find what I want, but I point it in the right direction and it sorts itself out), reading a codebase and summarizing it. Debugging too, though there it's most useful for the "what's the exact command again" moments when I'm too lazy to look it up.

Where's the delegate/keep line? I delegate most of the coding now - there's very little it won't do for me. What I keep: anything touching infra or live environments (I don't give it write access), the testing, and the calls that need my judgment. Design and architecture are a combo - my thinking, its writing and diagrams. Net, it does maybe 60-75% of what I used to do by hand.

## The parts that worry me

This is where I'm not fully at peace.

**The grind is what made me an engineer.** Beyond the usual worries - slop, outdated or unmaintained deps, security holes, code no human wants to maintain - my real concern is subtler. The daily grind, the bug chases, the digging: that's what builds the muscle memory that lets you *just know* where to look or how to write something. It's what turns into experience you can draw on later. AI quietly takes some of that away. When I sit down to actually code now, I often have to re-read syntax and docs because I've let it slip - or I never got as sharp in the first place (Rust, for me: about two years in, still not seasoned). Same with design: I used to build intuition from experimentation, from reading other people's war stories, from talking to the right people. Now the AI seems to have all the answers... until it doesn't, or it hands you the wrong one. My fear is that AI lets people skip the foundations and skip the experience of actually *being* an engineer. Yes, you naturally code less as you move up - if you don't want to be an IC forever, some fade is normal - but this feels different. (One genuinely good pairing, though: Rust + AI. The compiler is strict enough that it catches a lot of the AI's nonsense for you.)

**Review has gotten weird.** The thing that bugs me most: sometimes I review AI-written code and get AI-written *replies* back - so the person who opened the MR is basically a proxy, and might not fully understand what their own change does. That's frustrating. And there's an asymmetry - you can silently ignore an AI review comment (just resolve it), but you can't do that to a human. People seem more dismissive of AI feedback than a person's. It cuts the other way too: push back hard enough and the AI will fold and agree with you (some humans will too :)) ). Quality varies a lot: a local agent with real context can be genuinely useful (with the occasional *wtf are you talking about?!*), while a bot commenting on a bare MR diff tends to be shallow. It comes down to how much you feed it - the whole codebase, the architecture, the decisions vs. just the diff.

**Failure modes.** Because I review every command and every line before it runs, it rarely blows up badly enough to remember. The common stuff: it reaches for old API versions or unmaintained libraries (looking at you, `serde_yaml`), and I catch it and hand it a link or an alternative. A colleague once spent a good while in dependency hell because the AI kept using `azure_storage_blobs` instead of `azure_storage_blob` - one letter :sweat_smile: . And every so often you get the pure "this makes no sense at all" MR. (I've authored a few of those too, to be fair.)

## Leading a team through this

I lead a platform/enablement team, so this isn't just personal.

We're still building the processes and onboarding around AI - well-integrated tooling, better CI hooks (in a noisy pipeline with dozens of failures: which are because of *my* change, which are flakiness, which are a side effect of someone else's?). The goal is simple: make engineers as efficient as they can be. But I'm holding a hard line on one thing - people still need to understand their own changes and be able to debug something at an odd hour when it breaks, because AI isn't wired deeply enough into our infra to just "go fix it" for you. The foundations are a MUST.

Same reasoning on hiring. "If everyone's using AI, why still do technical assessments?" Because using AI doesn't remove the need to know how things are built. I think of it like a kid learning to read, write, and do basic math - fundamentals that carry you regardless of what tools show up later. Maybe that shifts over the next 10-100 years. For now, you still need them.

## Where I land

Net, I'm positive - with all of the above in mind. Learn the fundamentals anyway. Don't let the grind vanish entirely, because the grind *is* the training.

And on trust: right now I won't let an agent run or deploy to prod, or even staging - read-only access to staging, occasionally prod when I judge it safe, and that's it. That will change, but only with real guardrails: controls at the kernel level, not a polite instruction to "please don't do that" - because we've all watched it do exactly that more than once :) .

So yeah - AI writes a lot of my code and most of my words now, including these. I'll take the speed. I just refuse to trade away the part that made me an engineer to get it.
