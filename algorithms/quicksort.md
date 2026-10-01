---
type: Algorithm Profile
title: Quicksort
description: Behaviour, the pivot policy decision, the duplicate-key cliff, and measured step counts for the phase-1 quicksort.
tags: [algorithms, quicksort, pivot, worst-case, reference]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: sort-algo-wiki
    resource: https://en.wikipedia.org/wiki/Sorting_algorithm
    title: Sorting algorithm
  - id: quick-wiki
    resource: https://en.wikipedia.org/wiki/Quicksort
    title: Quicksort
  - id: dutch-flag
    resource: https://en.wikipedia.org/wiki/Dutch_national_flag_algorithm
    title: Dutch national flag algorithm
---

# Quicksort

Registry ID `quicksort`. In-place, **not stable**, O(log n) average stack. The algorithm the
registry is most careful about, because its cost depends on a design choice that is not visible
in the name.

Phase-1 configuration: **Lomuto partition with a median-of-three pivot**
([ADR-003](/decisions/adr-003-pivot-policy.md)).

## Mechanism

Choose a pivot, partition the array so that everything ≤ pivot lands left of it and everything
greater lands right, then recurse on both sides.

Quicksort's entire behaviour follows from *how the pivot is chosen*. Two implementations both
called "quicksort" can differ by a factor of five in comparisons on the same input. That is why
`pivot_policy` is a mandatory registry field
([FR-9](/requirements/functional-requirements.md)) and why it is echoed on every response and
stored on every result.

## The pivot trap

All figures at n = 32, identical inputs, only the pivot policy varying:

| Input | First/last element pivot | Median-of-three |
|---|---|---|
| Already sorted | **496** | 103 |
| Reverse sorted | **496** | 126 |
| 4 distinct values | 181 | 181 |
| All identical | **496** | **496** |

496 = `n(n−1)/2`, the quadratic worst case.

### Reading the table

**A naive pivot makes quicksort as slow as bubble sort on sorted input.** 496 comparisons,
identical to [bubble sort's worst case](/algorithms/bubble-sort.md). Sorting an array that is
*already in order* — the friendliest possible input — is quicksort's worst case, because every
pinned element is an extreme value and every partition peels off exactly one element.

**Median-of-three fixes the sorted and reverse-sorted cases**, cutting sorted input from 496 to
103 comparisons. It picks the middle of the first, middle, and last elements, which on a sorted
or reverse-sorted array is a near-median.

**Median-of-three fixes nothing for all-identical input.** Every pivot equals every other
element, so the partition cannot split: each pass pins one element and recurses on `n − 1`,
giving the full 496 comparisons *and* 1,116 moves — worse than bubble sort on both counts.

This is a real limitation of Lomuto partition, not an artefact of the pivot rule. Lomuto
partition handles equal keys by pushing them all to one side, so any partition of an
all-duplicate array is maximally unbalanced regardless of which element is chosen.

### Why the limitation is kept

Fixing it is a phase-2 item
([3-way partition](/roadmap/future-phases.md)), and leaving it is deliberate. Per
[US-3](/requirements/user-stories.md), a learner can discover the cliff themselves: submit
thirty-two copies of `7` and watch quicksort match bubble sort's worst case while merge sort
stays at 32 comparisons. An implementation whose failure mode is quietly patched teaches
nothing about why worst cases exist.

The standard fix is a 3-way (Dutch national flag) partition, which splits elements into
`< pivot`, `= pivot`, and `> pivot` and recurses only on the outer two groups. Equal keys then
cost no recursion at all.[^dutch-flag]

## Complexity, given the stated pivot policy

| Case | `comparisons` | Class | `moves` |
|---|---|---|---|
| Best | ~`0.9·n log n` | O(n log n) | ~`n log n` |
| Average (random) | ~`1.39·n log n` | O(n log n) | ~`n log n` |
| Worst | `n(n−1)/2` | **O(n²)** | O(n²) |

Verified on random input (mean of 20 seeds): 103 comparisons at n = 32 on sorted input, 115 on
random input, 126 on reverse-sorted input, 496 on all-duplicate input. The first three are close
together; the last is 5.4× the average.

That gap between 115 and 496 **is** the argument for publishing worst-case complexity. An
implementation quoting only its average would look perfectly safe.

## Measured step counts on random input

Mean of 20 seeds, uniformly random integers.

| n | `comparisons` | `moves` |
|---|---|---|
| 8 | 14 | 40 |
| 16 | 42 | 95 |
| 32 | 115 | 223 |
| 64 | 302 | 525 |
| 128 | 748 | 1,213 |
| 256 | 1,763 | 2,683 |
| 512 | 4,196 | 6,057 |

Quicksort leads merge sort on both metrics from n = 64 upward on random input. The gap is small —
at n = 512 it is 4196 vs 3964 comparisons, about 6% — which is worth stating plainly: **on random
input the two are close, and the interesting differences are at the edges**, not in the mean.

## Degenerate and instructive inputs, n = 32

| Input | `comparisons` | `moves` | `auxiliary_slots` |
|---|---|---|---|
| Already sorted | 103 | 162 | 0 |
| Reverse sorted | 126 | 240 | 0 |
| 4 distinct values | 181 | 438 | 0 |
| All identical | **496** | **1,116** | 0 |

The `moves` column is the more alarming one. On all-duplicate input quicksort performs 1,116
moves — nearly three times [bubble sort's](/algorithms/bubble-sort.md) 496 moves on the same
input, and more than merge sort's 320, while doing no useful work at all. Degradation is not
merely slower; it does strictly more writing.

## Recursion strategy

The phase-1 implementation recurses into the **smaller** partition and loops on the larger,
which bounds stack depth at O(log n) in the good case. It does not eliminate the O(n²) recursion
depth on adversarial input; it only avoids the stack overflow that naive recursion would
suffer. Median-of-three keeps measured depth at 3 for n = 32 on sorted input.

## Stability

`stable: false`. Partitioning swaps elements across the pivot, so equal keys can be reordered.

This is a **behavioural** difference, not just a cost one: for an input with duplicate keys,
merge sort and quicksort can return different orderings, and both are correct. A caller who
depends on the relative order of equal elements must use merge sort. This is the one respect in
which merge sort is strictly more capable.

## Implementation requirements

* **Mutates its argument** — partitioning is in place. Relies on the orchestrator's copy
  ([FR-4](/requirements/functional-requirements.md)).
* Pivot-selection comparisons **must** be counted
  ([Rule 1](/design/step-counting-semantics.md#rule-1--comparisons)). They are 3 of the 8
  comparisons in the [worked example](/api/worked-example.md) and omitting them would produce a
  number matching no other implementation.
* Counters reset per execution
  ([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)).
* `auxiliary_slots` is `0` — the recursion is on the stack, not in an array.
* `stable: false`, `in_place: true`, `auxiliary_space: "O(log n)"`.
* `pivot_policy: median_of_three` and the duplicate-key `known_limitations` entry are
  **mandatory** registry fields. The registry rejects a partition-based algorithm that omits
  `pivot_policy` ([the registry contract](/design/algorithm-registry.md#registry-integrity)).
* Handles n = 0 and n = 1 with zero comparisons and zero moves.

## A note on hand-verification

A manual trace of this algorithm does **not** reliably reconcile with the instrumented
implementation, and [the worked example](/api/worked-example.md#the-quicksort-trace-above-is-wrong-and-that-is-the-point)
documents that failure rather than hiding it. Quicksort figures in this bundle come only from
instrumented runs recorded in
[measurement provenance](/references/measurement-provenance.md).

## Known limitations

Published in the registry response
([`GET /v1/algorithms`](/api/run-retrieval-endpoint.md)):

1. **All-duplicate input degrades to O(n²).** 496 comparisons and 1,116 moves at n = 32,
   versus merge sort's 80 and 320. Median-of-three does not protect against this; the fix is a
   3-way partition.
2. **Worst case also triggers on sorted and reverse-sorted input if the pivot policy is ever
   changed** to first/last element — currently 496 each, reduced to 103 and 126 by the current
   policy.
3. **Not stable.** Equal keys may be reordered; use merge sort if relative order matters.

[^dutch-flag]: Dutch national flag algorithm — the standard three-way partition that makes
    equal-key groups cost no recursion.