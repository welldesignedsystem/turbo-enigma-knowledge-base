---
type: Reference
title: Glossary
description: Domain terms for the Algorithm Run Service, including the precise meaning of a step.
tags: [meta, glossary, terminology]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: okf-spec
    resource: https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md
    title: Open Knowledge Format specification v0.2
---

# Glossary

Terms are defined once here. Other documents link to these definitions rather than
restating them, because loose terminology is the fastest way for a knowledge base to become
self-contradictory.

## Step

The most overloaded word in this domain. This bundle uses it in exactly one sense:

> **A step is one comparison or one move.** A run reports `comparisons` and `moves`
> separately; a single scalar "number of steps" is a derived convenience value
> (`work_units`), not a measured quantity.

The word "step" is never used in these documents to mean a loop iteration, a function call,
a basic block, or an array index, because each of those is a different unit and produces
incomparable numbers. See [step-counting semantics](/design/step-counting-semantics.md) for
the normative counting rules.

## Comparison

One evaluation of a relational test between two keys, such as `a[j] > a[j+1]`. Counted as
one `comparison`. A test that returns true *and* proceeds to test again is two comparisons.

## Move

One write of a key value from one array slot to another. **A swap of two elements counts as
two moves**, because two writes occurred — one per destination slot.

Whether copying into an auxiliary buffer counts is fixed by
[step-counting semantics](/design/step-counting-semantics.md); for merge sort it does, twice
per merge.

## Work units

The derived scalar `work_units = comparisons + moves`. Provided because callers often want
one number. Its 1:1 weighting of a comparison against a move is **arbitrary** — the two
have different costs on real hardware. It is never authoritative and must never be used to
rank algorithms. See [ADR-001](/decisions/adr-001-metric-vector-over-scalar.md).

## Run

One submitted input executed against a selected set of algorithms, producing one
`Result` per algorithm plus the run's own metadata. A run is the unit of work and the unit
of billing if the service is ever metered. See
[the run execution model](/design/run-execution-model.md).

## Algorithm

A registered, invocable sorting implementation conforming to the
[algorithm registry](/design/algorithm-registry.md) contract. Phase 1 registers
[bubble sort](/algorithms/bubble-sort.md), [merge sort](/algorithms/merge-sort.md), and
[quicksort](/algorithms/quicksort.md).

## Metric

One measured quantity attached to a result: `comparisons`, `moves`, `elapsed_ns`,
`auxiliary_slots`. Metrics are a vector, not a scalar.

## Reference sort

The independently computed sorted copy of the input against which every algorithm's output
is checked before the run is allowed to succeed. A mismatch fails the run. The reference
sort itself is not registered as an algorithm and is not reported in results.

## Input length limit

The maximum array length the service accepts, default 10,000 elements. It exists to bound
the cost of [bubble sort](/algorithms/bubble-sort.md), which is quadratic and dominates the
run at roughly 50 million comparisons at the limit. See
[input validation](/design/input-validation.md).

## Best / average / worst case

The minimum, expected, and maximum cost over all input permutations of a given length.
"Average case" for sorting algorithms conventionally means the average over *uniformly
random* permutations, not the average over real-world data distributions — a distinction
worth stating because real-world data is rarely uniform. See
[the complexity reference](/algorithms/complexity-reference.md).

## In-place

An algorithm is in-place if it sorts using O(1) auxiliary array slots beyond the input.
Bubble sort and the quicksort variant in phase 1 are in-place; merge sort is not. This is
why merge sort's move count is comparatively high.

## Pivot

The element quicksort partitions around. Its selection policy is a first-order factor in
quicksort's measured comparison count — a naive policy costs a factor of ~5 on already
sorted input. See [ADR-003](/decisions/adr-003-pivot-policy.md) and the
[pivot trap](/algorithms/quicksort.md#the-pivot-trap).

## Degenerate input

An input that drives an average-case-efficient algorithm into its worst case. For the
phase 1 quicksort these are already-sorted, reverse-sorted, and all-duplicate arrays. See
[the pivot trap](/algorithms/quicksort.md#the-pivot-trap).

## Phase 1

The first deliverable: a synchronous service implementing only the sort-comparison run. See
[scope and phasing](/requirements/scope-and-phasing.md).

## Design intent

A statement about what the service is intended to do. Unverifiable until code exists.
Carries `status: draft`. Contrast [measurement](/README.md#the-one-rule-that-keeps-this-bundle-honest).

## Measurement

A number produced by running instrumented reference implementations under stated conditions.
Valid only for those implementations. Conditions are recorded in
[measurement provenance](/references/measurement-provenance.md).