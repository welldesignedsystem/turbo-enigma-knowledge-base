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
| `bubble_sort` | 15 | **16** | 31 | 0 |
| `merge_sort` | **10** | 32 | 42 | 6 |
| `quicksort` | 17 | 30 | 47 | 0 |

All three return `[1, 2, 3, 5, 8, 9]`.

### The interesting part

There is no single winner, and that is the point.

* **Fewest comparisons:** merge sort, 10 — then quicksort at 17, and bubble sort last at 15.
* **Fewest moves:** bubble sort, 16 — barely half of quicksort's 30.
* **Largest `work_units`:** quicksort, 47, on the *fewest* moves after bubble sort.
* **Highest `comparisons` + `moves` disagreement:** quicksort, 17 comparisons and 30 moves,
  because 9 of those 17 comparisons are pivot selection.

Bubble sort wins on moves and loses on comparisons. Merge sort wins on comparisons and is worst
on moves, because it pays for its speed with writes
([the copy-out and copy-back rule](/design/step-counting-semantics.md#rule-2--moves)).
Quicksort is second on both counts and still produces the largest `work_units` total of the
three — which is exactly why an advisory scalar built by adding two incommensurable numbers
should not be read as a ranking.

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

### Quicksort — 17 comparisons, 30 moves

Median-of-three pivot selection, then Lomuto partition. Note that the recursion covers
`a[0:6]`, `a[4:6]`, `a[0:3]` — three partitions, and **9 of the 17 comparisons are
pivot selection**.

#### Partition 1 — `a[0:6]` = `[5,3,8,1,9,2]`, `lo=0`, `hi=5`, `mid=2`

Pivot selection, 3 comparisons ([Rule 1](/design/step-counting-semantics.md#rule-1--comparisons)):

| Test | Result | Action |
|---|---|---|
| `a[2]=8 < a[0]=5` | ✗ | — |
| `a[5]=2 < a[0]=5` | ✓ | swap `a[5],a[0]` (+2) → `[2,3,8,1,9,5]` |
| `a[5]=5 < a[2]=8` | ✓ | swap `a[2],a[5]` (+2) → `[2,3,5,1,9,8]` |

Then the median is moved to `hi`: swap `a[2],a[5]` (+2) → `[2,3,8,1,9,5]`. **Pivot = 5**, at
`a[5]`.

**3 comparisons, 6 moves.** Partition sweep testing `a[j] <= 5` for every `j` in `0..4` — note
that a failed test still costs a comparison:

| j | Compare | Action |
|---|---|---|
| 0 | `2 <= 5` ✓ | swap `a[0],a[0]` (+2, a self-swap) |
| 1 | `3 <= 5` ✓ | swap `a[1],a[1]` (+2, a self-swap) |
| 2 | `8 <= 5` ✗ | — |
| 3 | `1 <= 5` ✓ | swap `a[2],a[3]` (+2) → `[2,3,1,8,9,5]` |
| 4 | `9 <= 5` ✗ | — |

Final swap `a[3],a[5]` (+2) → `[2,3,1,5,9,8]`. Pivot settles at index 3. Running: **8
comparisons, 14 moves**. Recurse on `a[0:3]` and `a[4:6]`; `a[4:6]` is the smaller side.

#### Partition 2 — `a[4:6]` = `[9,8]`

Pivot selection (3 comparisons): `a[4]=9 < a[4]=9` ✗; `a[5]=8 < a[4]=9` ✓ → swap (+2) →
`[2,3,1,5,8,9]`; `a[5]=9 < a[4]=8` ✗. Then median → `hi`: swap `a[4],a[5]` (+2) →
`[2,3,1,5,9,8]`. **Pivot = 8.**

Sweep over `j=4`: `9 <= 8` ✗ — 1 comparison. Final swap `a[4],a[5]` (+2) →
`[2,3,1,5,8,9]`. Running: **12 comparisons, 20 moves.**

#### Partition 3 — `a[0:3]` = `[2,3,1]`

Pivot selection (3 comparisons): `a[1]=3 < a[0]=2` ✗; `a[2]=1 < a[0]=2` ✓ → swap (+2) →
`[1,3,2,5,8,9]`; `a[2]=2 < a[1]=3` ✓ → swap (+2) → `[1,2,3,5,8,9]`. Then median → `hi`: swap
`a[1],a[2]` (+2) → `[1,3,2,5,8,9]`. **Pivot = 2.**

Sweep: `j=0`: `1 <= 2` ✓ → self-swap `a[0],a[0]` (+2); `j=1`: `3 <= 2` ✗ — 2 comparisons.
Final swap `a[1],a[2]` (+2) → `[1,2,3,5,8,9]`. Running: **17 comparisons, 30 moves.**

The remaining subarrays are all size 1 and cost nothing.

**Total: 17 comparisons (9 pivot selection + 8 partition sweep), 30 moves.**

## Why this section used to be wrong, and what replaced it

This trace previously reconciled to nothing. The published figure was 8 comparisons, the
hand-written trace above it produced 12, and the mismatch was documented as evidence that
"quicksort cannot be hand-verified".

That conclusion was wrong, and the real cause is more interesting: **8 was a defective
measurement.** The harness that produced it omitted pivot-selection comparisons — precisely the
error [Rule 1](/design/step-counting-semantics.md#rule-1--comparisons) exists to prevent. Once
those 9 comparisons were counted, the trace and the instrumented run agree exactly at 17 and 30,
and bubble sort's and merge sort's traces still reconcile.

Two lessons survive, and they are stronger than the original:

1. **A hand trace and a measurement must come from the same specified algorithm.** The
   disagreement was not that hand-tracing is hard; it was that the trace and the harness
   described two different algorithms. Any trace that does not reconcile is evidence of a
   specification defect somewhere, not of a limitation of tracing.
2. **Every number here comes from an instrumented run** of
   [the reference implementation](/references/measurement-provenance.md), and the golden-number
   tests in [phase 1](/roadmap/phase-1-sort-comparison.md) exist to catch exactly this class of
   defect — a harness that silently drops a category of comparisons produces plausible numbers
   that match no other implementation.

## Reproducing this run

Submit:

```json
{ "input": [5, 3, 8, 1, 9, 2] }
```

Expect `comparisons`/`moves` of `15`/`16`, `10`/`32`, `17`/`30` under
`step_counting_version: "v1"`. To confirm the counting semantics have not drifted, check
`step_counting_version` in the response before comparing figures.

## Contrasts worth running next

| Input | What it demonstrates |
|---|---|
| `[1,2,3,4,5,6,7,8]` | Bubble sort's best case with early exit: 7 comparisons, 0 moves |
| `[8,7,6,5,4,3,2,1]` | Bubble sort's worst case: 28 comparisons |
| 32 copies of `7` | [The pivot trap](/algorithms/quicksort.md#the-pivot-trap): quicksort 589 comparisons, 1,116 moves |
| `[1,…,32]` | Quicksort with median-of-three: 151 comparisons, versus 496 for a naive pivot |

The same input submitted twice **must** return identical `comparisons` and `moves`. If it does
not, either the counters are leaking across iterations
([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)) or an
algorithm received a stale buffer
([FR-4](/requirements/functional-requirements.md)) — the two defects this service's design
exists to make detectable.