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

Mean over uniformly random integers in `[1, 10⁶)` (400 seeds at n ≤ 32, fewer above).

| n | Bubble `cmp` | Bubble `mov` | Merge `cmp` | Merge `mov` | Quick `cmp` | Quick `mov` |
|---|---|---|---|---|---|---|
| 8 | 26 | 28 | 16 | 48 | 26 | 39 |
| 16 | 113 | 120 | 46 | 128 | 69 | 96 |
| 32 | 480 | 493 | 122 | 320 | 171 | 227 |
| 64 | 1,976 | 2,028 | 305 | 768 | 412 | 526 |
| 128 | 8,041 | 8,122 | 736 | 1,792 | 971 | 1,198 |
| 256 | 32,429 | 32,718 | 1,727 | 4,096 | 2,232 | 2,696 |
| 512 | 130,441 | 130,761 | 3,964 | 9,216 | 5,089 | 6,049 |

### What the growth ratios show

Doubling `n` multiplies bubble sort's comparisons by about **4**, and merge sort's and
quicksort's by about **2.1**. A factor of 4 per doubling is the signature of Θ(n²); a factor of
2 with a slowly rising multiplier is the signature of n log n. This is the empirical content
behind the class labels, and it is what a learner should check for themselves.

Bubble sort overtakes merge sort somewhere between n = 16 and n = 32 in this table, and never
recovers. At n = 32 bubble sort is already 4× merge sort on comparisons; at n = 512 it is 33×.

Quicksort's growth here is visibly *above* n log n — 5,089 comparisons at n = 512 against merge
sort's 3,964 — because every partition pays three pivot-selection comparisons
([Rule 1](/design/step-counting-semantics.md#rule-1--comparisons)). Median-of-three buys
robustness against adversarial input at a constant-factor cost on random input. That trade is
only visible if the pivot policy is published, which is why
[`pivot_policy` is mandatory](/design/algorithm-registry.md).

## Why a single "steps" column is impossible

The three algorithms win different metrics, and the winner changes with the metric. At n = 512,
uniformly random input:

| Metric | 1st | 2nd | 3rd |
|---|---|---|---|
| Fewest `comparisons` | **merge sort** (3,964) | quicksort (5,089) | bubble sort (130,441) |
| Fewest `moves` | **quicksort** (6,049) | merge sort (9,216) | bubble sort (130,761) |
| Fewest `work_units` (1:1) | **quicksort** (11,138) | merge sort (13,180) | bubble sort (261,202) |

Merge sort performs the fewest comparisons and the second fewest moves. Quicksort is second on
comparisons and first on both moves and `work_units`. Merge the three metrics into one column
and you have silently chosen which of these to privilege — and therefore chosen the winner.

This is the concrete justification for
[ADR-001](/decisions/adr-001-metric-vector-over-scalar.md).

Two patterns in the n = 512 row are worth noticing on their own:

* Bubble sort's `comparisons` (130,441) and `moves` (130,761) are nearly equal. On random input
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
| Already sorted | 31 | 0 | 80 | 320 | 151 |
| Reverse sorted | **496** | 992 | 80 | 320 | 177 |
| All identical | 31 | 0 | 80 | 320 | **589** |
| 4 distinct values | 451 | 336 | 116 | 320 | 265 |
| Random (mean) | 480 | 493 | 122 | 320 | 171 |

Two things to read off this table:

1. **Merge sort's `moves` do not move at all** — 320 on every input, for the structural reason
   in
   [Consequence 3](/design/step-counting-semantics.md#consequence-3--merge-sorts-move-count-has-no-clean-closed-form).
   Its comparisons do vary, 80 to 122, but never quadratically: worst and best case are both
   O(n log n). Note that "input-independent" is true of its moves and only loosely true of its
   comparisons
   ([Consequence 6](/design/step-counting-semantics.md#consequence-6--input-independent-is-true-of-merge-sorts-moves-and-false-of-its-comparisons)).
2. **Quicksort's comparison count ranges from 151 to 589 — a factor of nearly four on inputs of
   identical length.** Its average case is not a safe planning assumption, which is precisely
   why `worst` and `pivot_policy` are mandatory registry fields
   ([ADR-003](/decisions/adr-003-pivot-policy.md)).

The all-identical row is the limitation that median-of-three does **not** fix: quicksort spends
589 comparisons — quadratic, like bubble sort's worst case, and in fact worse than bubble's 496
— and also incurs 1,116 moves, where bubble sort on that input needs 0. See
[the pivot trap](/algorithms/quicksort.md#the-pivot-trap).

## The pivot policy, side by side

Quicksort at n = 32, two pivot policies on identical inputs:

| Input | First/last element pivot | Median-of-three |
|---|---|---|
| Already sorted | **496** | **151** |
| Reverse sorted | **496** | **177** |
| 4 distinct values | **196** | 265 |
| All identical | **496** | **589** |

496 = `n(n−1)/2` = the quadratic worst case. The naive policy makes quicksort exactly as bad as
bubble sort on sorted input. Median-of-three fixes sorted and reverse-sorted, is *worse* than
the naive policy on 4-distinct input, and fixes nothing at all for all-identical input.

The 4-distinct row is worth pausing on: median-of-three is not a uniform improvement. It trades
a large constant-factor win on adversarial shapes for a small constant-factor loss when several
distinct values are present, because every partition costs three pivot-selection comparisons.

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