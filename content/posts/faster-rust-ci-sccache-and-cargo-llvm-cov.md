---
title: "Our Rust CI got slow, so I measured where the time actually went"
description: "Splitting one opaque 'tests' job showed the tests weren't slow - compiling them twice was. Then sccache and cargo-llvm-cov cut the pipeline to about a third."
date: 2026-08-15T10:00:00+09:00
lastmod: 2026-08-15T10:00:00+09:00
draft: false
authors: ["rolandjitsu"]

tags: ["rust", "ci", "performance"]
categories: ["DevOps"]
series: []

toc:
  enable: true
---

Our Rust CI got slow. Not "grab a coffee" slow - "go do something else and hope it's green when you come back" slow. This is the story of how I got most of that time back, and the boring lesson underneath it: *measure before you optimize.*

## Why a slow pipeline actually hurts

For a while I didn't care that the Rust jobs were slow. There weren't many contributions, and the pipeline was fast enough for the pace we shipped at. Then that changed - a lot more code started flowing into the workspace, a good chunk of it [AI-generated]({{< ref "how-ai-changed-the-way-i-work" >}}), and the pipeline that was "fine" suddenly wasn't.

Here's why it matters now. An MR has to wait for a green pipeline before it can merge. And if you need to rebase before merging - which, with a busy `main`, you often do - *the whole thing runs again*. So every minute on the pipeline is a minute multiplied across every contributor, every rebase, every retry. Slow CI isn't an inconvenience; it's something everyone pays, all day.

So it became worth my time. :sweat_smile:

## The pipeline layout

The pipeline ran in stages. The two slowest Rust jobs were the tests and the benchmarks, which ran in *parallel*, and then a build job ran in a later stage. (Building Docker images for all the Rust services was the other big cost - that's a later post) Roughly:

```yaml
stages: [test, build]

rust-tests:
  stage: test
  script:
    - cargo hack test --each-feature --locked
    - cargo tarpaulin --tests --engine llvm --out xml

rust-bench:
  stage: test            # runs in parallel with rust-tests
  script:
    - cargo bench --workspace

rust-build:
  stage: build           # waits for the whole `test` stage
  script:
    - cargo build --workspace --release
```

The bulk numbers I started with:

| Job    | Time    |
| ------ | ------- |
| tests  | ~1700s  |
| bench  | ~600s   |
| build  | ~240s   |

Nearly half an hour on the critical path (the slow test job, then the build). My gut said "the test suite is too big". My gut was wrong.

## Step 0: measure, don't guess

Before touching anything, I wanted to know *where* the 1700s in the tests job actually went. That job was one opaque command, so I split it into its phases and timed each one with `date`:

```bash
# was one opaque command:
#   cargo hack test --each-feature --locked

# split it, and time each part:
t=$(date +%s); cargo hack test --each-feature --locked --no-run \
  && echo "### build test bins took $(($(date +%s)-t))s"

t=$(date +%s); cargo hack test --each-feature \
  && echo "### run tests took $(($(date +%s)-t))s"

t=$(date +%s); cargo tarpaulin --tests --engine llvm --out xml \
  && echo "### coverage took $(($(date +%s)-t))s"
```

The result was the whole reason this post exists:

```text
### build test bins took 1243s
### run tests took 118s
### coverage took 377s
```

*Read that again*. The tests themselves took **118 seconds**. Compiling them took **1243**. And then coverage - via [`cargo tarpaulin`](https://github.com/xd009642/tarpaulin) - spent another **377s**, most of it *recompiling the same code a second time* with its own instrumentation.

So the "tests are slow" story was a lie. Compilation was slow, and I was paying for it twice. That single measurement rerouted everything I did next. :exploding_head:

## Fix 1: cache the build (sccache)

Local development doesn't recompile the whole thing on every change - incremental builds are fast because most of the artifacts are already sitting there. CI throws all of that away and starts cold every run (at least that was the setup we had). So the no-brainer first move: give CI a compilation cache.

I used [`sccache`](https://github.com/mozilla/sccache) with a shared object-storage backend, so the cache survives between runs *and* is shared across jobs:

```yaml
variables:
  # enable sccache for rust:
  RUSTC_WRAPPER: sccache
  # disable incremental builds (sccache doesn't support it):
  CARGO_INCREMENTAL: 0
  # point sccache at shared storage (e.g. https://github.com/mozilla/sccache/blob/main/docs/Azure.md)
  # so the cache outlives a single run and is shared across jobs:
  SCCACHE_AZURE_CONNECTION_STRING: ...

after_script:
  - sccache --show-stats     # print the hit rate for every job
```

On a warm cache, the difference is significant:

| Job    | Before | Cold cache | Warm cache |
| ------ | ------ | ---------- | ---------- |
| tests  | ~1700s | 1500s      | ~630s      |
| bench  | ~600s  | 800s       | ~310s      |
| build  | ~240s  | 200s       | ~100s      |

Two things worth calling out here, because they're easy to miss:

- **Cold runs can be *slower*.** Look at bench: 600s -> 800s cold. On a cache miss you pay to compile *and* to write the result back. sccache is an investment that only pays off once the cache is warm - which, in a busy pipeline, is almost always.
- **The parallel job still benefits.** The bench job runs at the same time as tests, but sccache allows concurrent reads and writes, so bench happily reuses artifacts produced by other runs (and even in-flight ones). You don't need jobs to be sequential to share a cache.

And the hit rate, once warm, was almost comical. From one job's `sccache --show-stats`:

```text
$ sccache --show-stats
Compile requests                   2266
Compile requests executed          1870
Cache hits                         1861
Cache misses                          3
Cache hits rate                   99.84 %
Average compiler                  3.296 s
Average cache read hit            0.046 s
```

`3.296s` to compile something vs `0.046s` to pull it from the cache. That ratio *is* the win - do it a couple thousand times per run and you can see where the minutes went.

## Fix 2: stop compiling twice for coverage (tarpaulin -> cargo-llvm-cov)

Remember that coverage step eating 377s, mostly on a second compile? tarpaulin (at least the way we ran it) did its own instrumented build, separate from the test build. So the pipeline compiled the workspace once to run tests, then *again* to measure coverage. Wasteful.

The fix was to switch to [`cargo-llvm-cov`](https://github.com/taiki-e/cargo-llvm-cov), which instruments and runs the tests in a *single* pass - one build gives you both pass/fail and a coverage report:

```bash
# before: run tests, then a SECOND instrumented build+run just for coverage
cargo hack test --each-feature --locked
cargo tarpaulin --tests --engine llvm --out xml

# after: one instrumented run - tests AND coverage from a single build
cargo llvm-cov --each-feature --locked --lcov --output-path coverage.lcov
```

Collapsing the two passes into one, on a warm cache, brought the whole test-plus-coverage job down to around **612s** - from the ~1738s (1243 + 118 + 377) it was spending before.

### One honest caveat about the coverage number

When we switched tools, our reported coverage jumped from about **69% to 80%**. It would be nice to claim we wrote a pile of tests overnight, but no - *the number is tool-relative*. tarpaulin and llvm-cov instrument differently, and we also changed how results were aggregated across feature flags. Same code, different measuring stick, different number.

So flag this with your team before someone high-fives over a phantom 11-point gain. A coverage percentage is a measurement, not a fact of nature; change how you measure and it moves.

## Where it landed

Stitching the critical path together - the slow test job, then the build - on a warm cache:

- **Before:** ~max(1700, 600) + 240 = **~1940s** (roughly 32 minutes)
- **After:** ~max(612, 310) + 100 = **~712s** (roughly 12 minutes)

Call it ~2.5-3x faster on a warm cache, from two changes that added no cleverness to the code at all: cache the expensive step, and stop doing it twice.

## What I took away from it

1. **Measure before you optimize.** I was one gut-feeling away from "splitting up the test suite" - which would have done nothing, because the tests were 118s of a 1700s job. A quick change to add `date` pointed me at the real cost: compilation.
2. **"Tests are slow" usually means "compiling is slow."** In Rust especially, the build dominates. If a CI test job is dragging, time the `--no-run` build separately before you touch anything else.
3. **Cache the expensive step, and share the cache.** sccache on shared storage turns a cold compile into a near-instant cache read. Just know that cold runs pay a write penalty - it's a bet on warm ones.
4. **Don't compile the same code twice.** Running coverage as a second instrumented build was pure waste; `cargo-llvm-cov` folds it into the test run.
5. **A coverage % is tool-relative.** Switching coverage tools can move the number without changing a line of tested code. Don't read it as progress.

None of this was clever. It was just refusing to guess. The slowest part of my pipeline turned out to be something I'd never have found by staring at the test code - which is exactly why you measure first.

*This is part of a small "taming our CI" series - the [git caching proxy]({{< ref "caching-git-clones-across-a-slow-network" >}}) came out of the same effort to stop paying for the same work over and over.*
