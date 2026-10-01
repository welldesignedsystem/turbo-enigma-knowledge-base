---
type: Design Document
title: Timing measurement
description: How elapsed_ns is measured, why it cannot be used to rank algorithms, and what it is good for.
tags: [design, timing, benchmarking, method]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Timing measurement

The short version: **`elapsed_ns` is reported because callers ask for it, and it is not a
ranking.** This document explains why, so that nobody later promotes it into one.

## The rule

Per [FR-7](/requirements/functional-requirements.md), the service **MUST NOT** present a
"fastest to slowest" ordering derived from `elapsed_ns`, and every response **MUST** carry a
disclaimer saying so. The disclaimer is part of the response body, not just the docs — a
consumer who only sees the JSON must still encounter the warning.

## What is measured

* A **monotonic** clock, never a wall clock. A wall clock can jump backwards (NTP correction,
  leap seconds) and would produce a negative duration.
* Per iteration, around the algorithm call only. Excludes request parsing, validation, JSON
  serialisation, and the reference-sort computation
  ([the verification pass is not part of any algorithm's cost](/design/step-counting-semantics.md#rule-4--what-is-never-counted)).
* Reported as the **median** across `iterations` when `iterations > 1`
  ([FR-10](/requirements/functional-requirements.md)).

Median, not mean: a single garbage-collection pause or context switch would move a mean by a
large factor. Median, not minimum: the minimum reports the luckiest run rather than the
typical one, which is a *different* lie. With `iterations: 1` the median is just the single
sample, and the response should make the sample count visible so a consumer knows how thin
the statistic is.

## Why it cannot rank algorithms

Rank on `elapsed_ns` and you are ranking **the machine, the moment, and the service's own
concurrent load** at least as much as you are ranking the algorithms. The confounds, in
rough order of how badly they distort a ranking:

| Confound | Effect |
|---|---|
| **Clock resolution** | At small n the true duration can be below the clock's granularity, so the measurement quantises to zero or to one tick. A "2× slower" result can be one tick versus two. |
| **JIT warm-up** | A just-started runtime executes unoptimised code. Early iterations can be **orders of magnitude** slower. Without warm-up the first algorithm to run looks worst purely by being first. |
| **Garbage collection** | Merge sort allocates on every iteration; a GC cycle lands on one algorithm and not another. This biases against allocating algorithms, which is backwards — extra space is the *point* of merge sort. |
| **CPU frequency scaling and thermal throttling** | Absolute times drift over the life of a process. Two iterations ten minutes apart are not comparable. |
| **Concurrent load** | Other tenants on the same machine, other processes, the store write of a previous request. |
| **Memory layout and cache state** | Buffer alignment, page faults on first touch, and cache warmth from the previous algorithm's execution all differ per algorithm. |
| **Interpreter vs. compiled** | Decisive and frequently the largest effect of all. The algorithms cost the same *operations* in every language; they cost wildly different *nanoseconds*. |

That last row is the one that most often gets forgotten. The step counts in this bundle are
language-independent; `elapsed_ns` is not comparable even between two implementations of the
same algorithm in two languages.

## What timing *is* good for

Timing is not useless — it is only useless for cross-run and cross-machine comparison. It is
legitimate for:

* **Per-run, within-environment diagnostics.** "This particular run was slow" is a real
  observation about this process on this machine.
* **Magnitude.** Not "quicksort is 40× faster" but "this run spent essentially all its time
  in bubble sort" — which is true, obvious from the complexity classes, and useful for
  understanding the [input limit](/design/input-validation.md).
* **Sanity-checking the step counts.** If a run reports step counts implying ~10 ms of work
  and the measured time is ~5 s, something is wrong — a bug, or contention. The disagreement
  is diagnostic even though neither number should be published.

## Why the service does not just omit timing

Two reasons, both practical:

1. **Callers ask for it.** A service that returns only step counts looks incomplete to
   someone who wants to know about real cost.
2. **Omission invites substitution.** If the service returns no timing, motivated callers
   will compute their own — outside any controlled conditions, with no warm-up, and
   published without the caveats. Returning a labelled, caveated measurement is more honest
   than returning nothing.

## Warm-up, and why it cannot rescue this

`options.warmup_iterations` exists solely to give a JIT-compiled runtime time to compile, and
warm-up results are discarded entirely.

Even with warm-up, the fundamental problem is untouched: a measurement from machine A does not
transfer to machine B. Warm-up removes one confound from a list of seven. It does not make
timings portable, and the docs must not imply that it does.

## The step-count alternative

Every confound above is absent from a step count. `comparisons` and `moves` are functions of
the input array and the algorithm alone. They are:

* identical across machines, languages, and implementations that follow
  [the counting rules](/design/step-counting-semantics.md),
* exactly reproducible,
* unaffected by garbage collection, frequency scaling, or concurrent load,
* and the reason two people submitting the same array get the same answer
  ([US-1](/requirements/user-stories.md)).

This is the service's core bet: **step counts are the portable measurement; timings are
context.** Everything else in the design follows from it, including
[why the API returns a metric vector](/decisions/adr-001-metric-vector-over-scalar.md) rather
than a single number.

## If timings ever need to be trustworthy

Not phase 1, and not achievable by tuning the current approach. The requirements would be:

1. Dedicated measurement workers, one algorithm at a time, with pinned cores and a fixed
   `GOMAXPROCS`-equivalent.
2. Warm-up, then many iterations, reporting percentiles and a confidence interval.
3. Garbage collection paused or accounted for separately per sample.
4. The environment recorded alongside the result — model, clock resolution, runtime version —
   because a timing without its environment is not interpretable.
5. Published as a distribution, never as a point estimate.

Even then, the result describes that machine at that moment. Step counts remain the only
figures in this service that transfer. See [future phases](/roadmap/future-phases.md).