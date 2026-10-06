---
type: Decision Record
title: "ADR-003: Median-of-three pivot policy for quicksort"
description: Quicksort uses Lomuto partition with a median-of-three pivot. This does not fix the all-duplicate case, which is a deliberate phase-2 item.
tags: [adr, quicksort, pivot, worst-case, teaching]
status: accepted
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: dutch-flag
    resource: https://en.wikipedia.org/wiki/Dutch_national_flag_algorithm
    title: Dutch national flag algorithm
  - id: quick-wiki
    resource: https://en.wikipedia.org/wiki/Quicksort
    title: Quicksort
---

# ADR-003: Median-of-three pivot policy for quicksort

**Status:** accepted · **Date:** 2026-10-02 · **Deciders:** bundle author (agent-drafted,
unreviewed — see [trust and review](/README.md#trust-and-review))

## Context

Quicksort's comparison count depends far more on **how the pivot is chosen** than on anything
else about the implementation. Two implementations both named "quicksort" can differ by a factor
of five on the same input. Any specification of quicksort for this service must therefore fix the
policy, not just name the algorithm.

## Decision

The phase-1 quicksort uses **Lomuto partition with a median-of-three pivot**, selected from the
first, middle, and last elements by value and moved into the pivot slot.

`pivot_policy: median_of_three` is a **mandatory** registry field
([FR-9](/requirements/functional-requirements.md)), echoed on every response and stored on every
`Result` ([NFR-M4.2](/nfr/maintainability.md#nfr-m4--registry-changes-do-not-rewrite-history)).

## Evidence

Measured at n = 32 on identical inputs, only the pivot policy varying
([the pivot trap](/algorithms/quicksort.md#the-pivot-trap)):

| Input | First/last element pivot | Median-of-three |
|---|---|---|
| Already sorted | **496** | **151** |
| Reverse sorted | **496** | **177** |
| 4 distinct values | **196** | 265 |
| All identical | **496** | **589** |

496 = `n(n−1)/2`, the quadratic worst case. Note that median-of-three *loses* on the
4-distinct row: the pivot machinery is a cost, not only a benefit.

## Why median-of-three over the alternatives

| Policy | Behaviour | Rejected because |
|---|---|---|
| **First element** | 496 on sorted input | Catastrophic on the friendliest possible input. |
| **Last element** | 496 on sorted input | Identical failure to first-element, by symmetry. |
| **Random pivot** | Good expected behaviour | Unacceptable for a **reference** service. It would make `comparisons` non-deterministic, destroying the reproducibility that is the entire premise ([US-1](/requirements/user-stories.md)). |
| **Randomised + median-of-three** | Best average case | Same problem: no longer reproducible. Also unmeasurable by a learner. |
| **Median-of-three (chosen)** | 151 on sorted, 177 reverse, 589 on all-identical | Deterministic, and immune to the two most common adversarial shapes — but not free, and not a fix for duplicates. |

**Determinism is the deciding factor, not worst-case quality.** A pivot policy chosen at random
per call would produce a different comparison count for every run — so a learner's second run
would disagree with their first, and [the service's core promise](/requirements/product-brief.md)
would fail. Quicksort *implementations* commonly use randomisation; a quicksort *reference
service* must not.

Randomisation remains the textbook answer for production quicksort, where expected-case
performance across adversarial inputs matters more than reproducibility. That is a real
trade-off, resolved differently here on purpose.

## The limitation this decision does not fix

**Median-of-three does not protect against all-duplicate input.** At n = 32 it performs
**589 comparisons and 1,116 moves** — quadratic like the naive pivot's 496, and worse than
[bubble sort's](/algorithms/bubble-sort.md) worst-case comparison count, while doing no useful
work. (The figure is not exactly `n(n−1)/2` because each of the 31 partitions adds three
pivot-selection comparisons, which is itself part of the lesson below.)

The cause is Lomuto partition itself, not the pivot rule: Lomuto pushes all elements equal to
the pivot onto one side, so any partition of an all-duplicate array is maximally unbalanced
regardless of which element is selected as pivot. No pivot-selection policy fixes this. Only a
different partition scheme does.

**This limitation is kept deliberately.** Per
[US-3](/requirements/user-stories.md), a learner should be able to find the cliff themselves:
submit thirty-two copies of `7`, watch quicksort spend 589 comparisons and 1,116 moves, while
merge sort does the same work in 80 comparisons and 320 moves and
[bubble sort](/algorithms/bubble-sort.md) needs 0 moves — an array already in order — and work
out why. An implementation whose failure mode is quietly patched teaches nothing about why worst
cases exist.

The limitation is published, not hidden — in the registry's `known_limitations`, in
[the quicksort profile](/algorithms/quicksort.md#known-limitations), and in the
[API example](/api/run-retrieval-endpoint.md#why-pivot_policy-and-known_limitations-are-mandatory).

## Consequences

### Positive

* Immune to the two most common adversarial shapes (sorted, reverse-sorted) — the cases a
  beginner actually produces by typing arrays in order.
* Deterministic, so the service's reproducibility promise holds.
* Keeps the "quicksort is the fast one" intuition mostly intact, so the other lessons stay
  visible.

### Negative

* **The worst case is still reachable**, and reachable by a trivial input anyone can type. A
  caller who does not read `known_limitations` can hit O(n²) at any size.
* **The median-of-three overhead is much larger than it looks.** Three comparisons per partition,
  over 31 partitions at n = 32, is **93 of the 151 comparisons on sorted input — 62% of the
  total**. On the 4-distinct input it is a net loss against a naive pivot (265 against 196).
  This is the honest cost of the policy: it buys robustness against adversarial shape by paying
  a large constant factor on every partition.
* A fixed pivot policy is theoretically vulnerable to an adversary who knows the policy. Not a
  real concern for a teaching service, and recorded for completeness.

### Neutral

* The phase-2 3-way partition **will** change quicksort's measured counts on duplicate-heavy
  input. Per [NFR-M4](/nfr/maintainability.md#nfr-m4--registry-changes-do-not-rewrite-history),
  that requires bumping the published version and must not retroactively alter archived results.
  The 589-comparison golden number
  ([NFR-M2.3](/nfr/maintainability.md#nfr-m2--golden-numbers-guard-against-drift)) is a regression
  fixture *for this version*, not a permanent property of quicksort.

## How to tell this decision was wrong

If median-of-three's worst case proves unacceptable in practice, the options are the 3-way
partition (fixes duplicates, keeps determinism) or randomised pivot selection (better expected
behaviour, loses reproducibility). **The second is not available** without reopening
[ADR-001](/decisions/adr-001-metric-vector-over-scalar.md)'s premise, because a non-deterministic
`comparisons` value cannot be the service's authoritative result.

## Related

* [Quicksort](/algorithms/quicksort.md) — full profile and the pivot trap.
* [The pivot trap](/algorithms/quicksort.md#the-pivot-trap) — the measured comparison.
* [Algorithm registry](/design/algorithm-registry.md) — why `pivot_policy` is mandatory.
* [Future phases](/roadmap/future-phases.md) — the 3-way partition.