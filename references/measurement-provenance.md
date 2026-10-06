---
type: Reference
title: Measurement provenance
description: How every measured figure in this bundle was produced, and the strict limits of what those figures prove.
tags: [provenance, measurement, method, trust]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Measurement provenance

Every number in this bundle that is not a complexity class or a target came from running an
instrumented implementation. This document records what was run, under what conditions, and —
more importantly — what the results are **not** evidence for.

This exists because the bundle mixes design intent with measurement, and the two have very
different standing. See
[the bundle's honesty rule](/README.md#the-one-rule-that-keeps-this-bundle-honest).

## What was run

Instrumented **reference implementations** of the three phase-1 algorithms, written in Python
3 and executed to produce the figures under
[step-counting semantics `v1`](/design/step-counting-semantics.md):

* `bubble_sort` — with the early-exit optimisation, counting each inner comparison and 2 moves
  per swap.
* `merge_sort` — top-down, splitting at `mid = n // 2`, counting one comparison per merge-step
  test and 2 moves per element merged (copy out plus copy back), with size-1 leaves costing
  nothing.
* `quicksort` — Lomuto partition with **median-of-three** pivot selection, counting
  pivot-selection comparisons and 2 moves per swap.

These are throwaway verification scripts. **They are not part of this bundle and are not the
service.** The service does not exist yet
([scope and phasing](/requirements/scope-and-phasing.md)).

## The implementations under test satisfy these properties

* Each sort's output was asserted equal to the reference sort of the input, on every
  measurement. No figure below comes from a run that produced a wrong ordering.
* Quicksort, merge sort, bubble sort and a naive first-pivot quicksort were all checked for
  correctness across sorted, reverse-sorted, all-equal, few-distinct and randomly generated
  arrays at every length from 0 to 44: 0 mismatches. A correctness gate of this kind is
  mandatory — see [the correction record](#a-correction-to-these-figures) below for a case where
  an implementation that returned nonsense produced entirely plausible numbers.
* The counting rules were implemented directly from the normative text, including the
  swap-is-2-moves rule and the merge sort copy-out/copy-back rule.

## Figures produced, and where they appear

| Figure | Value | Appears in |
|---|---|---|
| Worked example, all three algorithms | 15/16, 10/32, 17/30 | [worked example](/api/worked-example.md) |
| Bubble sort worst case `comparisons` | `n(n−1)/2`, exact at n = 4, 8, 16, 32 | [step-counting semantics](/design/step-counting-semantics.md) |
| Bubble sort best case (early exit) | `n−1`, 0 moves | [step-counting semantics](/design/step-counting-semantics.md) |
| Merge sort `moves` | measured at n = 1…32, including non-powers of two | [step-counting semantics](/design/step-counting-semantics.md) |
| Merge sort comparison min/max | exhaustive over all permutations, n = 1…9 | [step-counting semantics](/design/step-counting-semantics.md) |
| Pivot policy comparison at n = 32 | 496 vs 151 sorted; 496 vs 177 reverse; 196 vs 265 4-distinct; 496 vs 589 all-equal | [quicksort](/algorithms/quicksort.md), [ADR-003](/decisions/adr-003-pivot-policy.md) |
| Growth table, random inputs | mean over 400 seeds (n ≤ 32) / fewer above, n = 8…512 | [complexity reference](/algorithms/complexity-reference.md) |
| Merge sort `comparisons`, random input | 122 at n = 32, 736 at n = 128 — **input-dependent**, unlike its `moves` | [Consequence 6](/design/step-counting-semantics.md#consequence-6--input-independent-is-true-of-merge-sorts-moves-and-false-of-its-comparisons) |
| Quicksort recursion depth, smaller-side-first | 8 at n = 32, 23 at n = 10,000 | [quicksort](/algorithms/quicksort.md#recursion-strategy) |

## How each figure was obtained

* **Growth table** — uniformly random integers in `[1, 10⁶)`, mean of 400 independent seeds per
  size for n ≤ 32 and 50–200 above. Uniform-random is the conventional meaning of "average case"
  ([glossary](/glossary.md#best--average--worst-case)); it is **not** a claim about any real
  data distribution.
* **Degenerate inputs** — single fixed arrays (all-sorted, reverse-sorted, all-equal,
  4-distinct) rather than samples, because the figures are exact properties of those inputs.
* **Merge sort bounds** — exhaustive enumeration over all permutations of distinct values for
  n ≤ 9. Permutations of distinct values are sufficient because comparison counts are invariant
  under relabelling; duplicate-containing inputs give different counts, which is exactly what
  [the pivot trap](/algorithms/quicksort.md#the-pivot-trap) is about for quicksort, and is
  covered by the separate fixed all-duplicate measurement at n = 32.
* **Pivot policy comparison** — three policies on identical fixed inputs at n = 32, so the
  only variable is the policy.

## What these figures are **not**

This is the part that matters most.

1. **Not evidence about the service.** The service does not exist. Every step count in this
   bundle describes a reference implementation, not the thing described by
   [the functional requirements](/requirements/functional-requirements.md).
2. **Not language-portable measurements.** The figures are Python timings-free step counts, so
   `comparisons` and `moves` are implementation-portable *by construction* — they are integer
   operation counts, not machine measurements. That portability is the entire premise of
   [the product brief](/requirements/product-brief.md). It holds only for implementations that
   follow [the counting rules](/design/step-counting-semantics.md) exactly. An implementation
   that counts pivot-selection comparisons differently will report different quicksort figures
   and will be wrong, not differently-wrong.
3. **Not a timing measurement, and no timing was performed.** No wall-clock figure appears
   anywhere in this bundle as a measurement. Every `elapsed_ns` value in
   [the API examples](/api/run-comparison-endpoint.md) is an explicitly labelled illustrative
   placeholder. See [timing measurement](/design/timing-measurement.md) for why.
4. **Not a proof of any complexity class.** The complexity classes in
   [the complexity reference](/algorithms/complexity-reference.md) are textbook results and
   come from published sources, cited per-algorithm. The measured figures are consistent with
   them and do two useful extra things: they give concrete magnitudes, and in two cases
   (**Consequences 3 and 4** in [step-counting semantics](/design/step-counting-semantics.md))
   they show that the textbook *closed forms* do not hold at non-power-of-two sizes. That is a
   correction to a common simplification, not a challenge to the asymptotic results.
5. **Not exhaustive for the quicksort figures.** Quicksort's 589/151/177/265 figures are single
   fixed inputs, not bounds. Only the merge sort min/max figures are exhaustive.
6. **Not stable across implementations.** Two implementations of "quicksort" with different
   pivot policies produce different numbers for the same input. That is why `pivot_policy` is a
   mandatory registry field
   ([ADR-003](/decisions/adr-003-pivot-policy.md)) and why `step_counting_version` is part of
   every response ([FR-8](/requirements/functional-requirements.md)).

## A correction to these figures

Every quicksort *comparison* figure in this bundle was wrong until an audit re-derived it. The
figures are recorded here rather than quietly replaced, because the failure is more instructive
than the numbers.

**What was wrong.** The bundle originally published 8 comparisons for quicksort on the worked
example, and 103 / 126 / 181 / 496 at n = 32. The correct figures under the specified
implementation are 17 and 151 / 177 / 265 / 589. The `496` figures for a *naive first-pivot*
quicksort were correct throughout — the two policies had been conflated.

**Why it happened.** The harness that produced the original numbers omitted quicksort's
pivot-selection comparisons — exactly the error
[Rule 1](/design/step-counting-semantics.md#rule-1--comparisons) exists to prevent. Every
**move** figure in the bundle was already correct, which is the tell: the bug was in counting
comparisons, not in the algorithm.

**The worst part.** The defective harness *also* did not sort correctly. Re-deriving the
figures required finding and fixing a bug in the measurement code itself (an index initialised
to the array offset rather than to 0), and a harness that produces wrong output will happily
report step counts for it. Nothing in a step count reveals that the underlying run was
meaningless.

**What changed in this bundle.** The reference implementations were rewritten with a
correctness gate — every implementation must reproduce the reference sort of every input at
every length from 0 to 44 before any figure is taken — and every numeric claim across the
bundle was re-derived from those gated implementations. One documented convention was added as
a result: [a self-swap counts 2 moves](/design/step-counting-semantics.md#rule-2--moves).

**What survived.** Bubble sort's figures were correct and are unchanged, including
`n(n−1)/2` at every length tested. Merge sort's `moves` were correct and are unchanged. Merge
sort's *comparisons*, however, had been presented as input-independent; they are not, and that
claim is now narrowed in
[Consequence 6](/design/step-counting-semantics.md#consequence-6--input-independent-is-true-of-merge-sorts-moves-and-false-of-its-comparisons).

**The generalisable lesson.** An agent-produced figure is a claim, not a measurement, until a
correctness gate stands in front of it. This is the strongest available argument for the
golden-number tests in [phase 1](/roadmap/phase-1-sort-comparison.md) and for the rule that no
figure enters this bundle without stated provenance: this bundle's own headline example
survived review precisely because someone re-derived it.

## The quicksort hand trace now reconciles

The earlier version of [the worked example](/api/worked-example.md) published a quicksort
hand-trace that matched nothing, and documented the mismatch as proof that quicksort cannot be
hand-verified. That conclusion was wrong: the *measurement* was defective, not the method. The
trace has been rewritten and reconciles exactly at 17 comparisons and 30 moves, and the lesson
retained is the stronger one — a trace and a measurement must describe the same algorithm, and
a disagreement between them means a specification defect, not an inherent limitation.

## Trust state of this document

This document is `unverified` in OKF terms: it records what an agent did, and no human has
confirmed it. That is the correct tier for the *report* of a measurement. It is **not** the
tier of the underlying measurements themselves — those are reproducible by re-running the
same reference implementations under the same stated conditions, and any future implementation
following `step_counting_version: "v1"` can check itself against the
[golden numbers](/api/worked-example.md).

That is the intended way to move this material up the trust ladder: implement the service,
check its output against the figures above, and record a `process:`-verified entry on the
concepts that then become confirmed. See
[the trust and review section](/README.md#trust-and-review).