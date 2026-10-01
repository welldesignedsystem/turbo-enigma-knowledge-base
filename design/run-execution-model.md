---
type: Design Document
title: Run execution model
description: How a run executes synchronously, the defensive-copy rule, and the verification gate.
tags: [design, execution, correctness, defensive-copy]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Run execution model

How a run turns one request into verified results. The mechanism is described step by step
in [the request lifecycle](/architecture/request-lifecycle.md); this document covers the three
decisions that mechanism embodies and why each one is easy to get wrong.

## Synchronous execution in phase 1

A run executes to completion on the request-handling goroutine/thread before the response is
written. See [ADR-002](/decisions/adr-002-synchronous-execution.md) for the full argument.

Concretely this means:

* There is no queue, no worker pool, no job state, and no polling.
* The only bound on execution is the per-run time budget
  ([FR-11](/requirements/functional-requirements.md)).
* `execution_mode` is reported as `"synchronous"` so a stored run remains interpretable after
  a future migration to `asynchronous`.

## Decision 1 — one defensive copy per execution

**Every** algorithm execution receives a freshly copied input array. Per algorithm, and per
iteration including warm-up.

This is a correctness requirement, not hygiene. The three phase-1 implementations are split
between in-place and out-of-place:

| Algorithm | Mutates its argument? |
|---|---|
| [Bubble sort](/algorithms/bubble-sort.md) | **Yes**, sorts in place |
| [Quicksort](/algorithms/quicksort.md) | **Yes**, partitions in place |
| [Merge sort](/algorithms/merge-sort.md) | No, builds a new array |

### Why the single-copy-per-algorithm version is not enough

The failure is worth tracing, because it is silent and it produces *plausible* numbers.

Registry order is `bubble_sort`, `merge_sort`, `quicksort`. With one copy made per
algorithm, reusing that copy across iterations:

1. Bubble sort receives `[5,3,8,1,9,2]` and sorts it in place. The buffer is now
   `[1,2,3,5,8,9]`.
2. Merge sort receives that same buffer. It performs **10 comparisons** — exactly the
   best-case count for a sorted array, not the average count.
3. Quicksort receives the same buffer. It performs **8 comparisons** — its *best* case, not
   the average.

Compare against the [measured figures](/api/worked-example.md) for the same input:
bubble 15, merge 10, quick 8. Interestingly the merge and quick numbers here coincide with
the correct averages, which is the trap: **nothing looks broken.** The comparison still
returns three algorithms' results, they are all correctly sorted, and the numbers are
plausible. But two of the three algorithms were measured on a different input than the one
the caller submitted, and the figure would be wrong for any other array.

The same argument applies across iterations: with `iterations: 100`, an in-place algorithm
run repeatedly on one buffer measures a sort, then 99 re-sorts of an already-sorted array.
`comparisons` would collapse toward `n−1` and `moves` toward 0. Nothing would flag it.

### What this makes true

Two properties the service depends on:

* **Order independence.** An algorithm's metrics depend on the submitted input and nothing
  else — not on which algorithms ran before it, not on how many times it ran.
* **Verifiability.** [FR-4](/requirements/functional-requirements.md) is testable by
  asserting that running all three algorithms yields the same per-algorithm metrics as
  running each in isolation. That test is in
  [the phase 1 definition of done](/roadmap/phase-1-sort-comparison.md).

Implementation note: the copy must be a **deep copy of the array of values**, and it must
not alias the caller's buffer or any previously handed-out buffer. A shallow copy of a slice
header, or a `copy()` into a reused scratch buffer without resetting its contents, is the
same bug wearing a disguise.

## Decision 2 — the verification gate

Before any result is returned, each algorithm's output is compared against an independently
computed [reference sort](/glossary.md#reference-sort) of the submitted input.

**Rules:**

1. The reference sort is computed **once per run**, before any algorithm executes, from the
   original submitted input.
2. Any mismatch **fails the entire run**. No partial results are returned
   ([FR-6](/requirements/functional-requirements.md)).
3. The reference sort is **not** one of the registered algorithms and does not appear in
   `results[]`. It is the yardstick, not a competitor.
4. The reference sort should be a **different implementation family** from the registered
   algorithms where practical, so a bug shared between two algorithms cannot confirm itself.

### Why fail-whole-run rather than flag-and-continue

Because a partial comparison is worse than an error. A response containing two of three
algorithms, with the third flagged, invites a consumer to read the two numbers as if they
were the result of a complete comparison. The failure is silent and it corrupts the
conclusion the service exists to support. An HTTP error is loud and correct.

### Why the reference sort is in-process

It could be delegated to an external sorting library or a remote service. Both are worse
here:

* A remote service adds a network call and a network failure mode to the one code path that
  must never be wrong, and cannot be right if it is unreachable.
* A built-in host-language sort is attractive for independence, but introduces a fourth
  silent behaviour: it may be unstable, may have implementation-defined ordering for equal
  keys, and almost certainly does not count steps the same way. That is acceptable for
  *ordering verification* and unacceptable for *step counting* — which is precisely why the
  reference sort is used for the first and never for the second.

## Decision 3 — sequential, deterministic execution order

Algorithms execute **sequentially in registry declaration order**, never concurrently.

Concurrent execution would be faster in wall-clock terms and is tempting because the
algorithms are independent. It is rejected for phase 1 for three reasons:

1. **The timing measurements would interfere with each other.** Two sorts running on
   different cores contend for memory bandwidth. Each `elapsed_ns` would be a measurement of
   contention rather than of the algorithm — and `elapsed_ns` is the one number here that is
   already the least trustworthy
   ([timing measurement](/design/timing-measurement.md)). Making it worse serves nobody.
2. **Order would stop being reproducible.** [FR-4](/requirements/functional-requirements.md)
   requires deterministic order so results are reproducible.
3. **It buys little.** At the [input limit](/design/input-validation.md) the whole run is a
   single slow [bubble sort](/algorithms/bubble-sort.md) regardless; parallelising merge sort
   and quicksort saves a few microseconds out of tens of milliseconds.

If parallel execution is ever added, the correct form is **isolated measurement workers**
running one algorithm at a time on dedicated cores, which is a phase-3 concern and would
still not make timings portable across machines.

## The reference run

The [worked example](/api/worked-example.md) is the canonical run for this service. It exists
so that every implementation of phase 1 can be checked against the same numbers, and any
disagreement is a detectable defect in the counting semantics or the algorithms rather than
an unremarkable difference.

| Algorithm | `comparisons` | `moves` |
|---|---|---|
| Bubble sort | 15 | 16 |
| Merge sort | 10 | 32 |
| Quicksort | 8 | 30 |

Input: `[5,3,8,1,9,2]`. Ruleset: [step-counting semantics `v1`](/design/step-counting-semantics.md).
Provenance: [measurement provenance](/references/measurement-provenance.md).