---
type: Product Brief
title: Product brief — Algorithm Run Service
description: Why an API for running algorithm comparisons exists, who uses it, and what it refuses to be.
tags: [product, vision, teaching]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Product brief — Algorithm Run Service

## The problem

Comparing sorting algorithms is a staple of every introductory algorithms course and every
interview prep page, and it is nearly always taught wrong in one of two ways:

1. **Complexity claims are quoted, not demonstrated.** "Merge sort is O(n log n)" is asserted
   from a textbook and never checked. A reader cannot tell whether their implementation
   actually behaves that way, or whether their intuition for why is correct.
2. **Timing is used as evidence.** Someone runs each algorithm on a laptop, sees that
   bubble sort took 900 ms and quicksort took 0.4 ms, and concludes quicksort is 2000×
   faster. That comparison is confounded by clock resolution, JIT warm-up, garbage
   collection, CPU frequency scaling, and background load — none of which say anything
   about the algorithms.

The underlying issue is that **step counts are deterministic and portable, and wall-clock
times are neither**, but almost nothing makes the first easy to obtain. Most tooling gives
you the second and implies it is the first.

## The idea

An HTTP API that accepts an array of integers, runs it through a set of registered sorting
algorithms under controlled conditions, and returns:

* the sorted output of **each** algorithm,
* a **metric vector** per algorithm (`comparisons`, `moves`, and where meaningful
  auxiliary space),
* an `elapsed_ns` measurement that is explicitly labelled as environment-dependent and
  unsuitable for ranking,
* a verification result confirming every algorithm produced the correct order.

Because comparisons and moves are properties of the algorithm and the input alone, two
people on two machines submitting the same array get the *same* step counts. That
reproducibility is the product. The timing is included because people ask for it, and is
labelled honestly rather than omitted — see [timing measurement](/design/timing-measurement.md).

## The first example: sort comparison

Phase 1 registers exactly three algorithms — [bubble sort](/algorithms/bubble-sort.md),
[merge sort](/algorithms/merge-sort.md), and [quicksort](/algorithms/quicksort.md) — chosen
because between them they expose the contrasts that matter:

| Algorithm | Best | Average | Worst | Space | In-place |
|---|---|---|---|---|---|
| Bubble sort | O(n) | O(n²) | O(n²) | O(1) | yes |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | O(n) | no |
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) | yes |

They span the entire interesting space: an algorithm whose cost depends on input *order*
(bubble), one that trades space for guaranteed time (merge), and one that is fast on
average but has an input-dependent cliff (quick). No other trio of three teaches as much.
See [the complexity reference](/algorithms/complexity-reference.md) for measured figures.

The quicksort in this trio also carries a deliberate, documented flaw. Its pivot policy
protects it against already-sorted input but **not** against all-duplicate input, which
still collapses to quadratic. That is documented as a known limitation rather than quietly
fixed, because a reference implementation whose failure mode is hidden teaches less than one
that names it. See [ADR-003](/decisions/adr-003-pivot-policy.md).

## Who it is for

* **Students and self-learners** who want to see step counts rather than take them on
  faith, and to feed in their own edge cases to find out where each algorithm breaks.
* **Instructors** who need a shared, reproducible reference: the same URL returns the same
  step counts for every student, so a disagreement becomes about the algorithm rather than
  about who has a faster laptop.
* **Engineers** building the algorithms themselves, who need a conformance harness that
  checks correctness and gives them a regression baseline.

## What it explicitly is not

* **Not a benchmarking platform.** It is not statistically rigorous. It does not do warm-up
  iteration, percentile reporting, or control-group CPU isolation well enough to publish a
  performance claim. Those belong in a future phase; see
  [future phases](/roadmap/future-phases.md).
* **Not a general algorithm-as-a-service.** "Submit any code" is a different and much larger
  security problem. Algorithms are compiled into the binary via the
  [registry](/design/algorithm-registry.md).
* **Not a source of truth for performance guarantees.** It measures what it can measure
  deterministically and tells you plainly what it cannot.
* **Not phase 1's job to be fast.** The input size limit exists to protect
  [bubble sort](/algorithms/bubble-sort.md), not to serve high-volume traffic. See
  [input validation](/design/input-validation.md).

## The core design position

The single most consequential choice in this service is that **it returns a metric vector,
not a scalar**. A single "number of steps" number is what the request for this service
literally asked for, and it is provided — as `work_units` — but it is derived and
arbitrarily weighted, and the docs say so every time it appears. The authoritative return
value is `{comparisons, moves}`. Reasoning is in
[ADR-001](/decisions/adr-001-metric-vector-over-scalar.md).