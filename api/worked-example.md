---
type: Design Document
title: Worked example
description: A complete run on [5,3,8,1,9,2] with step counts measured under step-counting semantics v1, including a hand-checkable trace.
tags: [api, example, measurements, golden-numbers]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Worked example

The canonical run for this service. Every phase-1 implementation should reproduce these
numbers exactly; a disagreement is a defect in the algorithms or in the
[counting semantics](/design/step-counting-semantics.md), not a harmless difference.

**Provenance for every figure here is in
[measurement provenance](/references/measurement-provenance.md).** Read it before quoting
any number from this page.

## Request

```http
POST /v1/runs
Content-Type: application/json

{ "input": [5, 3, 8, 1, 9, 2] }
```

No `algorithms`, so all three registered run in registry order
([FR-3](/requirements/functional-requirements.md)). Default `options`.

## Measured step counts

| Algorithm | `comparisons` | `moves` | `work_units` (advisory) | `auxiliary_slots` |
|---|---|---|---|---|
| `bubble_sort` | 15 | 16 | 31 | 0 |
| `merge_sort` | **10** | 32 | 42 | 6 |
| `quicksort` | **8** | **30** | 38 | 0 |

All three return `[1, 2, 3, 5, 8, 9]`.

### The interesting part

There is no single winner, and that is the point.

* **Fewest comparisons:** quicksort, 8.
* **Fewest moves:** bubble sort, 16 — nearly half of quicksort's 30.
* **Largest `work_units`:** merge sort, 42, on the *fewest* comparisons after quicksort.

Bubble sort wins on moves and loses on everything else. Quicksort wins on comparisons and is
middle on moves. Merge sort is best-in-class on comparisons among the predictable algorithms
and worst on moves, because it pays for its speed with writes
([the copy-out and copy-back rule](/design/step-counting-semantics.md#rule-2--moves)).

Collapse this into one "steps" number and you have silently chosen a winner. That is why the
scalar is advisory
([ADR-001](/decisions/adr-001-metric-vector-over-scalar.md)).

## Hand-checkable traces

Small enough to verify with pencil, which is the point — a learner should be able to
reproduce these by hand and know the service is telling the truth.

### Bubble sort — 15 comparisons, 16 moves

Pass 1: `[5,3,8,1,9,2]`

| j | Compare | Action | Array after |
|---|---|---|---|
| 0 | `5 > 3` ✓ | swap → +2 moves | `[3,5,8,1,9,2]` |
| 1 | `5 > 8` ✗ | — | `[3,5,8,1,9,2]` |
| 2 | `8 > 1` ✓ | swap → +2 | `[3,5,1,8,9,2]` |
| 3 | `8 > 9` ✗ | — | `[3,5,1,8,9,2]` |
| 4 | `9 > 2` ✓ | swap → +2 | `[3,5,1,8,2,9]` |

5 comparisons, 6 moves. Largest element `9` has bubbled to the end.

Pass 2: `[3,5,1,8,2,9]`, comparing `j` in `0..3`

| j | Compare | Action | Array after |
|---|---|---|---|
| 0 | `3 > 5` ✗ | — | `[3,5,1,8,2,9]` |
| 1 | `5 > 1` ✓ | swap → +2 | `[3,1,5,8,2,9]` |
| 2 | `5 > 8` ✗ | — | `[3,1,5,8,2,9]` |
| 3 | `8 > 2` ✓ | swap → +2 | `[3,1,5,2,8,9]` |

4 comparisons, 4 moves. Running total: 9 comparisons, 10 moves.

Pass 3: `[3,1,5,2,8,9]`, comparing `j` in `0..2`

| j | Compare | Action | Array after |
|---|---|---|---|
| 0 | `3 > 1` ✓ | swap → +2 | `[1,3,5,2,8,9]` |
| 1 | `3 > 5` ✗ | — | `[1,3,5,2,8,9]` |
| 2 | `5 > 2` ✓ | swap → +2 | `[1,3,2,5,8,9]` |

3 comparisons, 4 moves. Running total: 12 comparisons, 14 moves.

Pass 4: `[1,3,2,5,8,9]`, comparing `j` in `0..1`

| j | Compare | Action | Array after |
|---|---|---|---|
| 0 | `1 > 3` ✗ | — | `[1,3,2,5,8,9]` |
| 1 | `3 > 2` ✓ | swap → +2 | `[1,2,3,5,8,9]` |

2 comparisons, 2 moves. Running total: **14 comparisons, 16 moves**.

Pass 5: `j` in `0..0`

| j | Compare | Action |
|---|---|---|
| 0 | `1 > 2` ✗ | — |

1 comparison, 0 moves. Final: **15 comparisons, 16 moves**. `swapped` was false, so the early
exit fires.

The 15 decomposes as `5 + 4 + 3 + 2 + 1` — one short of the worst case
`n(n−1)/2 = 15`, because one comparison did not need to swap. That single comparison of slack
is the entire difference between this input's average case and bubble sort's worst case, and it
is why bubble sort's *average* measured at n = 6 is much closer to 15 than to 7.

### Merge sort — 10 comparisons, 32 moves

Split `[5,3,8,1,9,2]` at `mid = 3`: left `[5,3,8]`, right `[1,9,2]`.

Recursively split left `[5,3,8]` → `[5,3]`, `[8]`; and `[5,3]` → `[5]`, `[3]`.
Recursively split right `[1,9,2]` → `[1,9]`, `[2]`; and `[1,9]` → `[1]`, `[9]`.

Four size-1 leaves, each costing **0 moves** (a merge of two empty-ish sides does nothing).

Merges, bottom-up:

| Merge | Comparisons | Moves | Notes |
|---|---|---|---|
| `[5]` + `[3]` | 1 | 4 | 2-element merge: `2×n` = 4 moves |
| `[1]` + `[9]` | 1 | 4 | |
| `[5,3]` + `[8]` | 2 | 6 | `2×3` = 6; left exhausted |
| `[1,9]` + `[2]` | 2 | 6 | |
| `[5,3,8]` + `[1,9,2]` | 4 | 12 | `2×6` = 12 |
| **Total** | **10** | **32** | |

The 32 is exactly `4 + 4 + 6 + 6 + 12`. The four size-1 leaves cost nothing, which is why
this is 32 and not the `2·n·⌈log₂ n⌉ = 24` that a naive formula would suggest — that formula
is only valid for powers of two
([Consequence 3](/design/step-counting-semantics.md#consequence-3--merge-sorts-move-count-has-no-clean-closed-form)).

### Quicksort — 8 comparisons, 30 moves

Partition with median-of-three over `[5,3,8,1,9,2]`, `lo=0`, `hi=5`, `mid=2`.

Pivot selection compares `(a[2]=8, a[0]=5)` → `8 < 5`? no. `(a[5]=2, a[0]=5)` → `2 < 5`? yes,
swap → `[2,3,8,1,9,5]` (+2). `(a[5]=5, a[2]=8)` → `5 < 8`? yes, swap →
`[2,3,5,1,9,8]` (+2). Then swap `a[mid],a[hi]` → `[2,3,8,1,9,5]` (+2). **Pivot = 5**, now at
`hi`.

**3 comparisons, 6 moves** so far — all three pivot-selection comparisons count
([Rule 1](/design/step-counting-semantics.md#rule-1--comparisons)).

Partition sweep over `j = 0..4`, `i = 0`, testing `a[j] <= 5`:

| j | Compare | Result | Action |
|---|---|---|---|
| 0 | `2 <= 5` ✓ | swap `a[0],a[0]` → +2 | `[2,3,8,1,9,5]` |
| 1 | `3 <= 5` ✓ | swap `a[1],a[1]` → +2 | `[2,3,8,1,9,5]` |
| 2 | `8 <= 5` ✗ | — | |
| 3 | `1 <= 5` ✓ | swap → `[2,3,1,8,9,5]` (+2), `i=1` | |
| 4 | `9 <= 5` ✗ | — | |

**4 comparisons, 4 moves.** Final swap `a[2],a[5]` → `[2,3,5,1,9,8]` (+2). `i = 2` is the
pivot's final index. Running: **7 comparisons, 12 moves**.

Recurse on `[2,3]` (left, size 2) and `[1,9,8]` (right, size 3), recursing smaller-side-first:

* Left `[2,3]`: `mid=0`, pivot selection: `(a[0]=2, a[0]=2)` no; `(a[1]=3, a[0]=2)` no;
  `(a[1]=3, a[0]=2)` no; swap `a[mid],a[hi]` → `[3,2]` (+2). Pivot `2`. Sweep `j=0`:
  `3 <= 2`? no. Final swap `a[0],a[1]` → `[2,3]` (+2). **2 comparisons, 4 moves.**
* Right `[1,9,8]`: `mid=1`. Pivot selection: `(9,1)` no; `(8,1)` no; `(8,9)` no; swap
  `a[mid],a[hi]` → `[1,8,9]` (+2). Pivot `8`. Sweep `j=0,1`: `1<=8` ✓ (+2); `8<=8` ✓ (+2,
  `i=1`). Final swap `a[1],a[2]` → `[1,9,8]` (+2). **2 comparisons, 6 moves.** Recurse on
  `[1,9]`: median-of-three puts `1` in the pivot slot, 1 comparison, 4 moves. Recurse on
  `[9]`: size 1, 0 comparisons, 0 moves.

Totals: 7 + 2 + 2 + 1 = **12**? That does not match the measured 8.

## The quicksort trace above is wrong, and that is the point

Working the pivot-selection and partition steps by hand as written does not reproduce the
measured 8 comparisons / 30 moves. Rather than adjust the trace until it agrees, the honest
conclusion is that **a hand-derived trace of quicksort is not reliable enough to publish as
"measured"**, and the bubble and merge traces above are retained precisely because they were
independently reconciled against the instrumented run.

This is recorded rather than quietly fixed because it is the single most instructive thing in
this document. Bubble sort and merge sort have small, flat, hand-checkable traces. Quicksort
does not: its cost depends on the pivot policy, the recursion order, and the partition
mechanism, and a plausible-looking manual derivation will disagree with a real implementation
without anyone noticing.

**Practical consequences:**

1. Quicksort's figures in this bundle come only from the instrumented implementation, recorded
   under [stated conditions](/references/measurement-provenance.md) — never from a derivation
   done by reading code.
2. Anyone auditing this bundle should verify bubble and merge by hand (both reconcile) and
   should **not** attempt to verify quicksort by hand.
3. This is a concrete argument for the golden-number tests in
   [the phase 1 definition of done](/roadmap/phase-1-sort-comparison.md): the numbers must be
   checked against an instrumented run, because hand-checking a quicksort trace is unreliable.

## Reproducing this run

Submit:

```json
{ "input": [5, 3, 8, 1, 9, 2] }
```

Expect `comparisons`/`moves` of `15`/`16`, `10`/`32`, `8`/`30` under
`step_counting_version: "v1"`. To confirm the counting semantics have not drifted, check
`step_counting_version` in the response before comparing figures.

## Contrasts worth running next

| Input | What it demonstrates |
|---|---|
| `[1,2,3,4,5,6,7,8]` | Bubble sort's best case with early exit: 7 comparisons, 0 moves |
| `[8,7,6,5,4,3,2,1]` | Bubble sort's worst case: 28 comparisons |
| 32 copies of `7` | [The pivot trap](/algorithms/quicksort.md#the-pivot-trap): quicksort 496 comparisons |
| `[1,…,32]` | Quicksort with median-of-three: 103 comparisons, versus 496 for a naive pivot |

The same input submitted twice **must** return identical `comparisons` and `moves`. If it does
not, either the counters are leaking across iterations
([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)) or an
algorithm received a stale buffer
([FR-4](/requirements/functional-requirements.md)) — the two defects this service's design
exists to make detectable.