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
* Quicksort was checked for correctness across 400 randomly generated arrays with 1 to 6
  distinct values: 0 mismatches.
* The counting rules were implemented directly from the normative text, including the
  swap-is-2-moves rule and the merge sort copy-out/copy-back rule.

## Figures produced, and where they appear

| Figure | Value | Appears in |
|---|---|---|
| Worked example, all three algorithms | 15/16, 10/32, 8/30 | [worked example](/api/worked-example.md) |
| Bubble sort worst case `comparisons` | `n(n−1)/2`, exact at n = 4, 8, 16, 32 | [step-counting semantics](/design/step-counting-semantics.md) |
| Bubble sort best case (early exit) | `n−1`, 0 moves | [step-counting semantics](/design/step-counting-semantics.md) |
| Merge sort `moves` | measured at n = 1…32, including non-powers of two | [step-counting semantics](/design/step-counting-semantics.md) |
| Merge sort comparison min/max | exhaustive over all permutations, n = 1…9 | [step-counting semantics](/design/step-counting-semantics.md) |
| Pivot policy comparison at n = 32 | 496 vs 103 sorted; 496 vs 126 reverse; 181 vs 181 duplicate-heavy; 496 vs 496 all-equal | [quicksort](/algorithms/quicksort.md), [ADR-003](/decisions/adr-003-pivot-policy.md) |
| Growth table, random inputs | mean of 20 seeds per size, n = 8…512 | [complexity reference](/algorithms/complexity-reference.md) |

## How each figure was obtained

* **Growth table** — uniformly random integers in `[1, 10⁶)`, mean of 20 independent seeds per
  size. Uniform-random is the conventional meaning of "average case"
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
5. **Not exhaustive for the quicksort figures.** Quicksort's 496/103/126/181 figures are single
   fixed inputs, not bounds. Only the merge sort min/max figures are exhaustive.
6. **Not stable across implementations.** Two implementations of "quicksort" with different
   pivot policies produce different numbers for the same input. That is why `pivot_policy` is a
   mandatory registry field
   ([ADR-003](/decisions/adr-003-pivot-policy.md)) and why `step_counting_version` is part of
   every response ([FR-8](/requirements/functional-requirements.md)).

## One documented negative result

The quicksort hand-trace in [the worked example](/api/worked-example.md#the-quicksort-trace-above-is-wrong-and-that-is-the-point)
does not reconcile with the instrumented run, and has been left in place rather than
reconciled. The bubble sort and merge sort traces were each independently re-derived and do
reconcile.

The conclusion recorded in that document — that quicksort's counts cannot be reliably
hand-verified, so they must come from an instrumented run — is a finding about method, and it
is the reason the golden-number tests
([phase 1](/roadmap/phase-1-sort-comparison.md)) compare against instrumented output rather
than against hand-derived expectations.

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