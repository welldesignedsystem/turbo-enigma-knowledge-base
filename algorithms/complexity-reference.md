---
type: Reference
title: Complexity reference
description: Side-by-side complexity and measured step-count comparison for the three phase-1 sorting algorithms.
tags: [algorithms, complexity, measurements, comparison]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: clrs
    resource: "Cormen, Leiserson, Rivest & Stein, Introduction to Algorithms (4th ed.), MIT Press 2022, ISBN 978-0-262-04640-6"
    title: Introduction to Algorithms, 4th edition
  - id: sort-algo-wiki
    resource: https://en.wikipedia.org/wiki/Sorting_algorithm
    title: Sorting algorithm
---

# Complexity reference

## Textbook complexity classes

Complexity classes are published results.[^clrs] The measured figures below are observations of
the implementations specified in this bundle. Provenance:
[measurement provenance](/references/measurement-provenance.md).

| Algorithm | Best | Average | Worst | Space | In-place | Stable |
|---|---|---|---|---|---|---|
| [Bubble sort](/algorithms/bubble-sort.md) | O(n) | O(n²) | O(n²) | O(1) | yes | yes |
| [Merge sort](/algorithms/merge-sort.md) | O(n log n) | O(n log n) | O(n log n) | O(n) | no | yes |
| [Quicksort](/algorithms/quicksort.md) | O(n log n) | O(n log n) | **O(n²)** | O(log n) | yes | **no** |

Read the stable column carefully: [quicksort is not stable](/algorithms/quicksort.md#stability),
and that is a behavioural property of the result, not merely of its cost. With duplicate keys,
two implementations can produce different — both correct — orderings.

## Measured growth, random inputs

Mean of 20 seeds per size, uniformly random integers in `[1, 10⁶)`.

| n | Bubble `cmp` | Bubble `mov` | Merge `cmp` | Merge `mov` | Quick `cmp` | Quick `mov` |
|---|---|---|---|---|---|---|
| 8 | 25 | 29 | 15 | 48 | 14 | 40 |
| 16 | 114 | 120 | 45 | 128 | 42 | 95 |
| 32 | 479 | 482 | 121 | 320 | 115 | 223 |
| 64 | 1,956 | 1,968 | 305 | 768 | 302 | 525 |
| 128 | 8,057 | 8,103 | 737 | 1,792 | 748 | 1,213 |
| 256 | 32,385 | 33,210 | 1,728 | 4,096 | 1,763 | 2,683 |
| 512 | 130,511 | 131,765 | 3,964 | 9,216 | 4,196 | 6,057 |

### What the growth ratios show

Doubling `n` multiplies bubble sort's comparisons by about **4**, and merge sort's and
quicksort's by about **2.1**. A factor of 4 per doubling is the signature of Θ(n²); a factor of
2 with a slowly rising multiplier is the signature of n log n. This is the empirical content
behind the class labels, and it is what a learner should check for themselves.

Bubble sort overtakes merge sort somewhere between n = 16 and n = 32 in this table, and never
recovers. At n = 32 bubble sort is already 4× merge sort on comparisons; at n = 512 it is 33×.

## Why a single "steps" column is impossible

The three algorithms win different metrics, and the winner changes with the metric. At n = 512,
uniformly random input:

| Metric | 1st | 2nd | 3rd |
|---|---|---|---|
| Fewest `comparisons` | **merge sort** (3,964) | quicksort (4,196) | bubble sort (130,511) |
| Fewest `moves` | **quicksort** (6,057) | merge sort (9,216) | bubble sort (131,765) |
| Fewest `work_units` (1:1) | **quicksort** (10,253) | merge sort (13,180) | bubble sort (262,276) |

Merge sort performs the fewest comparisons and the second fewest moves. Quicksort is second on
comparisons and first on both moves and `work_units`. Merge the three metrics into one column
and you have silently chosen which of these to privilege — and therefore chosen the winner.

This is the concrete justification for
[ADR-001](/decisions/adr-001-metric-vector-over-scalar.md).

Two patterns in the n = 512 row are worth noticing on their own:

* Bubble sort's `comparisons` (130,511) and `moves` (131,765) are nearly equal. On random input
  it performs almost exactly one swap per comparison, so its move count carries little
  information beyond its comparison count.
* Merge sort's `moves` (9,216) are roughly 2.3× its `comparisons` (3,964). That gap is the
  copy-out and copy-back cost
  ([Rule 2](/design/step-counting-semantics.md#rule-2--moves)) — the price merge sort pays for
  its guaranteed time.

## Degenerate inputs

Where the classes above stop being averages. All measured at n = 32:

| Input | Bubble `cmp` | Bubble `mov` | Merge `cmp` | Merge `mov` | Quick `cmp` (median-of-3) |
|---|---|---|---|---|---|
| Already sorted | 31 | 0 | 80 | 320 | 103 |
| Reverse sorted | **496** | 992 | 80 | 320 | 126 |
| All identical | 31 | 0 | 80 | 320 | **496** |
| 4 distinct values | 451 | 336 | 116 | 320 | 181 |

Two things to read off this table:

1. **Merge sort's comparison count barely moves** — 80 to 116 across every input shape, because
   its worst case and best case are the same O(n log n). Its `moves` do not move either, for the
   structural reason in
   [Consequence 3](/design/step-counting-semantics.md#consequence-3--merge-sorts-move-count-has-no-clean-closed-form).
2. **Quicksort's comparison count ranges from 103 to 496 — a factor of nearly five on inputs of
   identical length.** Its average case is not a safe planning assumption, which is precisely
   why `worst` and `pivot_policy` are mandatory registry fields
   ([ADR-003](/decisions/adr-003-pivot-policy.md)).

The all-identical row is the limitation that median-of-three does **not** fix: quicksort pays
the full 496 = `n(n−1)/2`, the same as bubble sort's worst case, and also incurs 1,116 moves.
See [the pivot trap](/algorithms/quicksort.md#the-pivot-trap).

## The pivot policy, side by side

Quicksort at n = 32, two pivot policies on identical inputs:

| Input | First/last element pivot | Median-of-three |
|---|---|---|
| Already sorted | **496** | 103 |
| Reverse sorted | **496** | 126 |
| 4 distinct values | 181 | 181 |
| All identical | **496** | **496** |

496 = `n(n−1)/2` = the quadratic worst case. The naive policy makes quicksort exactly as bad as
bubble sort on sorted input. Median-of-three fixes sorted and reverse-sorted, and fixes
nothing at all for all-identical input.

## Closed forms that do not hold everywhere

Two textbook shortcuts are exact only for power-of-two lengths, verified by exhaustive
enumeration up to n = 9 and by direct measurement to n = 32
([Consequences 3 and 4](/design/step-counting-semantics.md#consequence-3--merge-sorts-move-count-has-no-clean-closed-form)):

| Claim | Exact when | Fails when |
|---|---|---|
| Merge sort `moves` = `2·n·log₂ n` | `n` is a power of two | n = 3 → 10 not 12; n = 5 → 24 not 30; n = 9 → 58 not 72; n = 17 → 140 not 170 |
| Merge sort worst-case `comparisons` = `n·⌊log₂ n⌋ − n + 1` | `n` is a power of two | n = 3 → 3 not 1; n = 9 → 21 not 19 |

The merge sort `moves` formula overstates, because size-1 subtrees perform no merge. The
worst-case `comparisons` formula **understates** at n = 9, which makes the algorithm look
better than it is. Bubble sort's `n(n−1)/2` worst case, by contrast, is exact at every `n`
tested.

## The honest summary

There is no fastest sorting algorithm. There is a distribution of input shapes, and for each
one a different algorithm behaves differently. This service exists to make that observable —
which is why the input limit exists to protect the quadratic algorithm
([input validation](/design/input-validation.md)) and why the response is a
[metric vector](/decisions/adr-001-metric-vector-over-scalar.md) rather than a verdict.

[^clrs]: Cormen et al., *Introduction to Algorithms*, 4th ed. Establishes the complexity
    classes above. The measured figures are this bundle's own; see
    [measurement provenance](/references/measurement-provenance.md).