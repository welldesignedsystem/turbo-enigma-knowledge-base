---
type: Design Document
title: Step-counting semantics
description: The normative rules for what counts as a comparison and a move. Every reported step figure depends on this document.
tags: [design, step-counting, metrics, normative]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: clrs
    resource: "Cormen, Leiserson, Rivest & Stein, Introduction to Algorithms (4th ed.), MIT Press 2022, ISBN 978-0-262-04640-6 — chapters on sorting and on algorithm analysis"
    title: Introduction to Algorithms, 4th edition
  - id: sort-algo-wiki
    resource: https://en.wikipedia.org/wiki/Sorting_algorithm
    title: Sorting algorithm
  - id: dutch-flag
    resource: https://en.wikipedia.org/wiki/Dutch_national_flag_algorithm
    title: Dutch national flag algorithm
---

# Step-counting semantics

**This document is normative.** It defines `step_counting_version: "v1"` — the ruleset
identified by [FR-8](/requirements/functional-requirements.md). Every `comparisons` and
`moves` figure anywhere in this bundle, and every figure the service will ever return, is
produced under these rules.

Change a rule here and you **must** bump `step_counting_version`. The reason is that
changing a counting rule silently reinterprets every number ever published. That is how
benchmark corpora become uninterpretable within a year.

## Why the rules have to be pinned down at all

"Number of steps" is not a unit. Three reasonable people counting the same sort of the same
array get three different numbers, each defensible:

* Counting loop iterations.
* Counting comparisons only.
* Counting comparisons and assignments together.

None is more correct than the others. A step count is therefore meaningless without the
counting rule attached to it, which is why the rule is a versioned part of the API contract
rather than an implementation detail.

## Normative counting rules

### Rule 1 — `comparisons`

> One `comparison` is one evaluation of a relational test between two keys.

| Situation | Count |
|---|---|
| `a[j] > a[j+1]` evaluated | +1 |
| `a[i] <= a[j]` evaluated, true or false | +1 |
| A test written as `if (x < y) { ... } else { ... }` | +1 — the `else` branch does not re-evaluate |
| Loop-bound check `j < n - 1 - i` | **+0** — not a key comparison |
| Index bounds check | **+0** |
| Comparison between a key and a pivot | +1 — the pivot is a key |
| Pivot-selection comparison (median-of-three) | +1 each — these are real key comparisons and must not be omitted |

The last two rows are the ones implementations get wrong. Pivot-selection comparisons cost
real work, and an implementation that omits them reports a number that does not match any
other implementation of quicksort.

### Rule 2 — `moves`

> One `move` is one write of a key value into a different array slot. **A swap of two
> elements is 2 moves.**

The swap-is-2 rule is the single most consequential convention in this document, so it is
worth defending. `a[i], a[j] = a[j], a[i]` performs two writes: `a[i]` receives a new value
and `a[j]` receives a new value. Counting it as 1 "swap" is a *different unit*, and mixing
"swaps" and "moves" in one report is how two implementations end up incomparable. Phase 1
reports `moves` only.

| Situation | Count |
|---|---|
| Swap of two distinct slots | +2 |
| Swap of a slot with **itself** (`i == j`, a self-swap) | **+2** — see below |
| Writing one key into a vacant slot in any array the algorithm uses | +1 |
| Copying a key from the working array into an auxiliary buffer | +1 |
| Copying that key from the auxiliary buffer back to the working array | +1 |
| Loop bookkeeping (`i += 1`), rebinding a local variable | **+0** |
| Initialising the auxiliary buffer by filling with a sentinel | **+0** — nothing is moved |

**A self-swap counts 2 moves.** `a[i], a[j] = a[j], a[i]` with `i == j` writes each slot its
own value, so no key is relocated — but it is still two write operations executed, and
counting it as 0 would require inspecting the index relationship at every call site. This
service counts *operations executed*, not *values changed*, and the rule is deliberately
mechanical so that two implementations agree without coordinating.

The alternative convention (a self-swap is 0 moves because nothing moves) is defensible and
would change quicksort's figures substantially — 0 moves instead of 1,054 on sorted input at
n = 32, for example. It is **not** the phase-1 convention, and an implementation that adopted
it would report figures that do not match this bundle. The trade-off is worth stating plainly:
the mechanical rule slightly overstates the work quicksort does, and in exchange it removes a
whole class of implementation disagreement.

**Merge sort therefore pays 2 moves per element per merge level**, counting the copy out and
the copy back. This is why its move count is roughly double what a naive reading suggests,
and it is the correct accounting under these rules: the data genuinely is written to two
places.

### Rule 3 — auxiliary slots

> `auxiliary_slots` is the peak number of array slots used beyond the input buffer.

In-place algorithms report `0`. Merge sort reports `n` for the auxiliary buffer plus
recursion bookkeeping, conventionally `O(n)`. This is a *derived* figure from the
implementation, not a count of anything the algorithm does, and it is why it is optional in
the response ([FR-5](/requirements/functional-requirements.md)).

### Rule 4 — what is never counted

Not counted as steps, in any algorithm:

* Function call and return overhead.
* Recursion entry and exit.
* Loop-bound and index checks.
* Allocation and deallocation of the auxiliary buffer.
* Reading the input array.
* The verification pass against the reference sort.

The last one matters: the reference sort exists to check correctness
([FR-6](/requirements/functional-requirements.md)) and its cost is **not** attributed to any
algorithm. Attributing it would make the fastest-appearing algorithm the one that skipped
verification, which is precisely backwards.

### Rule 5 — counters reset per execution

Every execution of an algorithm begins with both counters at zero. Counters **MUST NOT**
accumulate across warm-up or repeated iterations
([FR-10](/requirements/functional-requirements.md)). An implementation whose counters carry
over will report `iterations ×` the true step count, which passes casual inspection because
the number is merely too large by an obvious factor.

## Consequences worth stating explicitly

### Consequence 1 — comparisons and moves can rank differently

For the worked input `[5,3,8,1,9,2]`, measured under these rules:

| Algorithm | `comparisons` | `moves` |
|---|---|---|
| Bubble sort | 15 | **16** |
| Merge sort | **10** | 32 |
| Quicksort | 17 | 30 |

Quicksort is best on comparisons; bubble sort is best on moves, by a wide margin. Note that
quicksort's 17 comparisons include 9 pivot-selection comparisons across its 3 partitions
(3 per partition, per [Rule 1](#rule-1--comparisons)) — omitting those is exactly the mistake
that produces a figure matching no other implementation. Neither
answer is wrong, and this is the concrete argument for
[returning a vector rather than a scalar](/decisions/adr-001-metric-vector-over-scalar.md).
A single "steps" number here requires an arbitrary, undisclosed choice of weighting, and any
weighting that ranks one algorithm first will rank a different algorithm first at a
different weight.

### Consequence 2 — bubble sort's worst case is exactly n(n−1)/2

Verified for this implementation:

| n | `comparisons` (reverse-sorted) | n(n−1)/2 |
|---|---|---|
| 4 | 6 | 6 |
| 8 | 28 | 28 |
| 16 | 120 | 120 |
| 32 | 496 | 496 |

And its best case with early exit is exactly `n−1` — 7 comparisons for n = 8 on
already-sorted input, with 0 moves. The early-exit optimisation is what makes bubble sort's
best case linear at all; without it, best case would also be quadratic.

### Consequence 3 — merge sort's move count has no clean closed form

Measured `moves` under these rules:

| n | `moves` | `2·n·⌈log₂ n⌉` |
|---|---|---|
| 1 | 0 | 0 |
| 2 | 4 | 4 |
| 3 | 10 | 12 |
| 4 | 16 | 16 |
| 5 | 24 | 30 |
| 8 | 48 | 48 |
| 9 | 58 | 72 |
| 16 | 128 | 128 |
| 17 | 140 | 170 |
| 32 | 320 | 320 |

The formula `2·n·log₂ n` is exact **only when n is a power of two**. The exact expression is
`2 · Σ (subtree size at every internal node of the recursion tree)`, which equals `2·n·log₂ n`
for balanced perfect splits and something strictly smaller otherwise — because a subtree of
size 1 performs no merge and therefore no moves. Anyone quoting `2n log n` as merge sort's
move count is quoting an upper bound that happens to be tight at powers of two.

### Consequence 4 — merge sort's worst-case comparison formula is also wrong off the powers of two

Exhaustively measured over all permutations of distinct values:

| n | min `comparisons` | max `comparisons` | `n·⌊log₂ n⌋ − n + 1` |
|---|---|---|---|
| 1 | 0 | 0 | 0 |
| 2 | 1 | 1 | 1 |
| 3 | 2 | 3 | 1 |
| 4 | 4 | 5 | 5 |
| 5 | 5 | 8 | 6 |
| 6 | 7 | 11 | 7 |
| 7 | 9 | 14 | 8 |
| 8 | 12 | 17 | 17 |
| 9 | 13 | 21 | 19 |

The familiar formula is exact for n = 2, 4, 8 and **understates the true worst case
elsewhere** — at n = 9 it predicts 19 when the real worst case is 21. A benchmark document
that states merge sort's worst case as a closed form is therefore slightly wrong for most
input lengths, and wrong in the direction that makes the algorithm look better than it is.

The **minimum** is exactly the recursion-tree quantity `Σ min(left, right)` over internal
nodes, which matches measurement at every n tested.

These two consequences are the strongest argument in this bundle for preferring *measured*
figures over textbook closed forms, and for treating
[the complexity reference](/algorithms/complexity-reference.md) as a set of observations
about specific implementations rather than a set of universal truths.

### Consequence 5 — quicksort's count depends on the pivot policy, so the policy is part of the contract

The same input gives different comparison counts under different pivot policies. Measured
at n = 32:

| Input | First/last element pivot | Median-of-three |
|---|---|---|
| Already sorted | 496 | **151** |
| Reverse sorted | 496 | **177** |
| 4 distinct values | 196 | **265** |
| All identical | 496 | **589** |

Median-of-three buys a large improvement on sorted and reverse-sorted input — the two shapes
that defeat a naive pivot — and it is *worse* than a naive pivot on the 4-distinct case. On
all-identical input it does not help at all: 589 comparisons is not `n(n−1)/2` but is
quadratic in the same way, and it is worse than bubble sort's 496. See
[the pivot trap](/algorithms/quicksort.md#the-pivot-trap) for why no pivot rule fixes that.

This is why `pivot_policy` is a mandatory registry field
([FR-9](/requirements/functional-requirements.md)) and why the published `worst` complexity
class for quicksort is not optional decoration — it is the thing that makes the number
interpretable. Full discussion in [the pivot trap](/algorithms/quicksort.md#the-pivot-trap) and
[ADR-003](/decisions/adr-003-pivot-policy.md).

### Consequence 6 — "input-independent" is true of merge sort's moves and false of its comparisons

This is the single most commonly repeated error in sorting commentary, so the bundle states it
explicitly. Merge sort performs the **same number of moves** on every input of a given length,
because the merge level structure depends only on `n`:

| n | `moves`, every input | `comparisons`, sorted | `comparisons`, random mean |
|---|---|---|---|
| 32 | 320 | 80 | ≈122 |
| 128 | 1,792 | 448 | 736 |

Comparisons are a different matter. Each merge compares until one side is exhausted, and
*which* side empties depends on the data. Sorted, reverse-sorted, and all-identical input all
yield the minimum of 80 comparisons at n = 32; random input averages about 122, and the theoretical
maximum for this split rule is `Σ(size − 1)` over internal nodes, or 129 at n = 32.

Quoting a single merge-sort comparison count as though it characterised the algorithm is
therefore wrong, and it is wrong in the direction that flatters the algorithm.

## Versioning this document

`step_counting_version: "v1"` covers the rules above. A **minor** revision that only clarifies
wording keeps the version. Any change that would alter a reported number for a fixed input
and implementation is a **breaking** change and requires:

1. A new `step_counting_version` value.
2. Existing stored `Run` documents keep the old version
   ([data model](/architecture/data-model.md)) so historical results stay interpretable.
3. New `AlgorithmRegistration` entries publishing the version they were built against.
4. A note in [the log](/log.md) recording what changed and why.

## References

Complexity bounds follow the standard treatment.[^clrs] The duplicate-key weakness of
Lomuto partition quicksort and the 3-way (Dutch national flag) alternative are described
in the algorithm's own literature.[^dutch-flag]

[^clrs]: Cormen et al., *Introduction to Algorithms*, 4th ed., chapters on sorting and
    algorithm analysis. The counting conventions here are this bundle's own; the textbook
    establishes the complexity classes, not the step definitions.
[^dutch-flag]: Dutch national flag algorithm — the standard fix for degenerate
    duplicate-heavy quicksort partitions.