---
type: Algorithm Profile
title: Bubble sort
description: Behaviour, measured step counts, and the input-shape dependence of the phase-1 bubble sort implementation.
tags: [algorithms, bubble-sort, reference]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: sort-algo-wiki
    resource: https://en.wikipedia.org/wiki/Sorting_algorithm
    title: Sorting algorithm
  - id: bubble-wiki
    resource: https://en.wikipedia.org/wiki/Bubble_sort
    title: Bubble sort
---

# Bubble sort

Registry ID `bubble_sort`. In-place, stable, O(1) auxiliary space. The simplest of the three
and the one that bounds [the input length limit](/design/input-validation.md).

## Mechanism

Repeatedly walk the unsorted prefix comparing adjacent elements and swapping them out of order.
After pass `i`, the largest `i` elements have bubbled to the end, so the unsorted prefix shrinks
by one.

The phase-1 implementation includes the **early-exit** optimisation: if a full pass performs no
swap, the array is sorted and the loop stops. This is the only reason bubble sort's best case is
linear.

## Complexity

| Case | `comparisons` | Class | `moves` |
|---|---|---|---|
| Best (sorted, early exit) | `n − 1` | O(n) | 0 |
| Average (random) | ≈ `n²/4` | O(n²) | ≈ `n²/4` |
| Worst (reverse sorted) | `n(n−1)/2` | O(n²) | `n(n−1)` |

Worst case verified exact at every `n` tested:

| n | `comparisons` (reverse sorted) | `n(n−1)/2` |
|---|---|---|
| 4 | 6 | 6 |
| 8 | 28 | 28 |
| 16 | 120 | 120 |
| 32 | 496 | 496 |

Best case: 7 comparisons and 0 moves at n = 8 on already-sorted input; 31 comparisons and 0
moves at n = 32.

## Measured step counts on random input

Mean over uniformly random integers (400 seeds at n ≤ 32, fewer above).

| n | `comparisons` | `moves` |
|---|---|---|
| 8 | 26 | 28 |
| 16 | 113 | 120 |
| 32 | 480 | 493 |
| 64 | 1,976 | 2,028 |
| 128 | 8,041 | 8,122 |
| 256 | 32,429 | 32,718 |
| 512 | 130,441 | 130,761 |

`comparisons` and `moves` track each other almost exactly, because on random input a comparison
results in a swap slightly more often than not. Bubble sort's move count therefore carries
almost no information its comparison count does not already carry — unlike merge sort, where
[the two diverge substantially](/algorithms/merge-sort.md#measured-step-counts-on-random-input).

## Degenerate and instructive inputs, n = 32

| Input | `comparisons` | `moves` | Why |
|---|---|---|---|
| Already sorted | 31 | 0 | Early exit after one pass. |
| All identical | 31 | 0 | No pair is out of order, so the first pass exits. |
| 4 distinct values | 451 | 336 | |
| Reverse sorted | **496** | 992 | Every comparison swaps; nothing is ever in place. |

`comparisons = 31` on both sorted and all-identical input is the same case: for bubble sort
those inputs are indistinguishable, because it only ever asks "is this pair out of order?"

## The hand-checkable trace

[the worked example](/api/worked-example.md#bubble-sort--15-comparisons-16-moves) walks
`[5,3,8,1,9,2]` pass by pass and reconciles to 15 comparisons and 16 moves. The 15 decomposes
as `5 + 4 + 3 + 2 + 1`, which sums to **exactly** the worst case `n(n−1)/2 = 15`. Not one
short of it: on this input bubble sort pays its full worst-case comparison bill.

What *is* below worst case is the move count. Reverse-sorted input of the same length performs
the same 15 comparisons but swaps every one of them, costing **30 moves** against this input's
16. The single comparison of slack is a comparison, and a comparison that finds a pair already
ordered costs no move — so the slack shows up entirely in moves.

That is a useful illustration that "average case" is an average over *all* permutations, not a
bound on any particular one, and that the two metrics can disagree about which input is
harder.

## Why it is in the registry

1. **It defines the input limit.** At n = 10,000 it performs 49,995,000 comparisons
   ([verified](/design/step-counting-semantics.md#consequence-2--bubble-sorts-worst-case-is-exactly-nn12)).
   The service's cost model is bubble sort's cost model.
2. **It is the only phase-1 algorithm whose cost depends on input *order*.** Merge sort and
   quicksort both run in the same ballpark on sorted and unsorted input; bubble sort runs 31
   comparisons on sorted input and 496 on reverse-sorted input — a factor of 16. That contrast is
   [US-2](/requirements/user-stories.md) in miniature.
3. **It is the only one with a linear best case**, which shows that an optimisation can change
   the complexity class of one case.

## Implementation requirements

* **Mutates its argument.** The orchestrator guarantees a fresh copy
  ([FR-4](/requirements/functional-requirements.md)); this algorithm uses it in place and
  returns the same buffer.
* Early-exit flag resets per execution
  ([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)).
* `auxiliary_slots` is `0`.
* `stable: true`. Equal elements are only swapped when strictly out of order, so relative order
  is preserved.
* Handles n = 0 and n = 1 with zero comparisons and zero moves.

## Known limitations

`known_limitations` is empty for this algorithm — a deliberate claim, not a default. Bubble sort
has no input that defeats it: it is O(n²) on *every* input and correct on all of them. That is
a genuine property, and it is the reason it is a useful yardstick against which quicksort's
input-dependent behaviour can be seen
([the pivot trap](/algorithms/quicksort.md#the-pivot-trap)).