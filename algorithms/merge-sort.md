---
type: Algorithm Profile
title: Merge sort
description: Behaviour, measured step counts, and the move-count cost of the phase-1 merge sort implementation.
tags: [algorithms, merge-sort, reference]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: sort-algo-wiki
    resource: https://en.wikipedia.org/wiki/Sorting_algorithm
    title: Sorting algorithm
  - id: merge-wiki
    resource: https://en.wikipedia.org/wiki/Merge_sort
    title: Merge sort
---

# Merge sort

Registry ID `merge_sort`. **Not** in-place, stable, O(n) auxiliary space. The only phase-1
algorithm whose cost is guaranteed on every input.

## Mechanism

Top-down: split at `mid = n // 2`, sort each half recursively, then merge the two sorted halves
by repeatedly taking the smaller front element.

The merge step is the only non-trivial part. Each merge-step test is one `comparison`, and
producing the merged output writes every element once — which under
[the move rules](/design/step-counting-semantics.md#rule-2--moves) means **two** moves per
element: one copying out of a half, one writing into the merged array.

## Complexity

| Case | Class | `comparisons` | `moves` |
|---|---|---|---|
| Best | O(n log n) | `Σ min(left, right)` over internal nodes | `2 · Σ (subtree size)` |
| Average | O(n log n) | — | — |
| Worst | O(n log n) | `n·⌊log₂ n⌋ − n + 1` **only for powers of two** | same as best |

## Merge sort's move count is identical on every input

Measured at n = 32:

| Input | `comparisons` | `moves` |
|---|---|---|
| Already sorted | 80 | 320 |
| Reverse sorted | 80 | 320 |
| All identical | 80 | 320 |
| 4 distinct values | 116 | 320 |
| Random (mean of 400 seeds) | ≈122 | 320 |

`moves` is **identical at 320 for every input**, because `moves` depends only on the shape of the
recursion tree, not on the data.

`comparisons` is **not** input-independent — a common and persistent error. It ranges from 80 to
about 122 on random input, with a theoretical maximum of 129 for this split rule, so a factor of
about 1.6. That is still far less variation than bubble sort (16×, from 31 to 496) or quicksort
(3.9×, from 151 to 589) on the same inputs, and it never becomes quadratic — but "the same speed
on every input" is true of merge sort's **moves** and only loosely true of its comparisons. See
[Consequence 6](/design/step-counting-semantics.md#consequence-6--input-independent-is-true-of-merge-sorts-moves-and-false-of-its-comparisons).

This is the algorithm's real selling point and the reason it is in the registry
([US-2](/requirements/user-stories.md)): it is the only algorithm in the trio whose **worst case
is its best case in moves**, and whose worst case is at most 1.6× its best in comparisons. Any
quadratic input-dependent blowup in a comparison of this trio is attributable to one of the
other two.

## The textbook closed form is wrong off the powers of two

Verified by exhaustive enumeration over all permutations of distinct values, n ≤ 9:

| n | min `comparisons` | max `comparisons` | `n·⌊log₂ n⌋ − n + 1` |
|---|---|---|---|
| 1 | 0 | 0 | 0 |
| 2 | 1 | 1 | 1 |
| 3 | 2 | 3 | **1** |
| 4 | 4 | 5 | 5 |
| 5 | 5 | 8 | **6** |
| 6 | 7 | 11 | **7** |
| 7 | 9 | 14 | **8** |
| 8 | 12 | 17 | 17 |
| 9 | 13 | 21 | **19** |

The familiar formula is exact at n = 2, 4, 8 and **understates the true worst case elsewhere** —
at n = 9 it predicts 19 when the real worst case is 21. A document claiming merge sort's worst
case as a closed form is therefore slightly optimistic for most input lengths.

The **minimum** is exactly `Σ min(left, right)` over the recursion tree's internal nodes, which
matches measurement at every `n` tested.

The same off-by-power-of-two caveat applies to the `moves` count:

| n | `moves` measured | `2·n·⌈log₂ n⌉` |
|---|---|---|
| 3 | 10 | 12 |
| 5 | 24 | 30 |
| 9 | 58 | 72 |
| 17 | 140 | 170 |
| 2, 4, 8, 16, 32 | matches exactly | — |

`2·n·⌈log₂ n⌉` overstates, because a subtree of size 1 performs no merge and costs no moves.
The exact figure is `2 · Σ (subtree size at every internal node)`. See
[Consequence 3](/design/step-counting-semantics.md#consequence-3--merge-sorts-move-count-has-no-clean-closed-form).

## Measured step counts on random input

Mean of 20 seeds, uniformly random integers.

| n | `comparisons` | `moves` | `moves` / `comparisons` |
|---|---|---|---|
| 8 | 15 | 48 | 3.2 |
| 16 | 45 | 128 | 2.8 |
| 32 | 121 | 320 | 2.6 |
| 64 | 305 | 768 | 2.5 |
| 128 | 737 | 1,792 | 2.4 |
| 256 | 1,728 | 4,096 | 2.4 |
| 512 | 3,964 | 9,216 | 2.3 |

The ratio settles around 2.3 as `n` grows. That is the copy-out/copy-back cost appearing
directly in the numbers: merge sort performs comparatively few tests and comparatively many
writes. It is the mirror image of [bubble sort](/algorithms/bubble-sort.md), whose two metrics
are nearly identical.

The ratio declines with `n` because `moves` grows as `2·n·log₂ n` while `comparisons` grows as
`n·log₂ n − O(n)`; the additive `−O(n)` in the comparison count becomes negligible.

## The hand-checkable trace

[the worked example](/api/worked-example.md#merge-sort--10-comparisons-32-moves) merges
`[5,3,8,1,9,2]` bottom-up and reconciles to 10 comparisons and 32 moves. The 32 decomposes as
`4 + 4 + 6 + 6 + 12`, with the four size-1 leaves costing nothing — which is exactly why the
total is 32 rather than the 24 a naive `2·n·⌈log₂ n⌉` would suggest.

## Implementation requirements

* **Does not mutate its argument** in the sense that matters: it builds and returns a new
  array. It is the one phase-1 algorithm that would tolerate a shared buffer, and it must not
  rely on that — every algorithm gets its own copy
  ([FR-4](/requirements/functional-requirements.md)).
* Counters reset per execution
  ([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)).
* `auxiliary_slots` is `n` (O(n)) for the working buffer plus recursion bookkeeping.
* `stable: true`. The merge test uses `<=` on the left operand, so ties resolve to the left half
  and relative order is preserved. Using `<` here would silently make merge sort unstable —
  a one-character difference with a behavioural consequence.
* Handles n = 0 and n = 1 with zero comparisons and zero moves.

## Known limitations

`known_limitations` is empty. Merge sort is O(n log n) on every input and is correct on all of
them. The cost is space, and that cost is already published as
`auxiliary_space: "O(n)"` and `in_place: false` rather than being a limitation of
correctness.

The real caveat is operational rather than algorithmic: merge sort allocates on every
invocation, which makes it disproportionately exposed to garbage collection in a timing
measurement ([timing measurement](/design/timing-measurement.md)). That is a measurement
artefact, not a property of the algorithm, and it is a good example of a confound that would
otherwise make merge sort look worse than it is.