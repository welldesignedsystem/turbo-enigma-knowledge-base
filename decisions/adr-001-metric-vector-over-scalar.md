---
type: Decision Record
title: "ADR-001: Return a metric vector, not a scalar step count"
description: The API returns {comparisons, moves} as the authoritative result. A single scalar steps number is derived and advisory.
tags: [adr, metrics, step-counting, api-design]
status: accepted
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# ADR-001: Return a metric vector, not a scalar step count

**Status:** accepted · **Date:** 2026-10-02 · **Deciders:** bundle author (agent-drafted,
unreviewed — see [trust and review](/README.md#trust-and-review))

## Context

The service was specified as returning "the number of steps" for each algorithm. That is a
reasonable-sounding requirement and it is what most comparisons of this kind publish.

The problem is that "steps" is not a unit, and collapsing several genuine measurements into one
number requires choosing a weighting between them. See
[step-counting semantics](/design/step-counting-semantics.md) for why the term is unusable
without a pinned rule.

## Decision

The authoritative per-algorithm result is a **metric vector**:

```json
{ "comparisons": 8, "moves": 30, "elapsed_ns": 971, "auxiliary_slots": 0 }
```

A scalar `work_units = comparisons + moves` **is** returned, because callers want one number and
the requirement asked for it — but it is explicitly derived, explicitly labelled advisory, and
explicitly not authoritative ([FR-5](/requirements/functional-requirements.md)).

Every place `work_units` appears states that its 1:1 weighting is arbitrary. The response
`disclaimer` and this ADR are the two places that must never be deleted.

## Why the scalar cannot be authoritative

Measured on identical inputs, different metrics produce different winners. At n = 512,
uniformly random input
([the complexity reference](/algorithms/complexity-reference.md#why-a-single-steps-column-is-impossible)):

| Metric | Winner |
|---|---|
| Fewest `comparisons` | **merge sort** (3,964) |
| Fewest `moves` | **quicksort** (6,057) |
| Fewest `work_units` | **quicksort** (10,253) |

Merge sort wins one metric and loses two. Any single-number summary must either discard a
metric or apply a weighting, and the weighting — not the measurement — decides the answer.

This is visible even at n = 6 on the
[worked example](/api/worked-example.md#the-interesting-part), where quicksort leads on
comparisons (8) and moves (30) but [bubble sort](/algorithms/bubble-sort.md) is far ahead on
moves (16).

### Any weighting will be arbitrary, and will be read as authoritative anyway

A 1:1 weighting of a comparison against a move is not derived from anything — the two have
different costs on real hardware, and no hardware-independent justification for their ratio
exists. It is a convenience. The failure mode is not that it is wrong; it is that it is
**read as if it were measured**, because it is a bare number in a JSON field with no
provenance.

Returning it under its real name, beside the vector it derives from, is the best available
mitigation. The alternatives were worse:

* **Omit the scalar.** Callers compute `comparisons + moves` themselves, which makes the
  weighting visible in *their* code rather than inherited invisibly from ours. This is
  actually recommended to clients, and documented as such.
* **Return several weighted composites** (`comparisons_only`, `moves_only`, …). Multiplies the
  number of numbers a caller must understand without removing the arbitrariness.
* **Return a normalised score.** Worst of the options. It looks authoritative, hides the
  inputs it was computed from, and actively encourages the
  [rejected "score out of 10" non-story](/requirements/user-stories.md#non-stories).

## Consequences

### Positive

* The response is honest about what was measured and what was chosen.
* A caller can build any ranking it likes and state its own weighting.
* Adding a metric — `auxiliary_slots`, `max_recursion_depth` — does not break the contract or
  require rescaling an existing number. This is why quicksort's `auxiliary_slots` could be
  added later without anyone having to recalibrate a composite score.

### Negative

* The contract is more complex than one integer.
* Some callers will read `work_units` and ignore the warning.
* The "steps" language of the original requirement is not literally satisfied, and that needs
  saying out loud rather than glossing.

### Neutral

* `results[]` is **never** sorted by any metric
  ([run comparison endpoint](/api/run-comparison-endpoint.md#ordering-guarantees)). A
  metric-ordered array would imply a winner that the data does not support.

## How to tell this decision was wrong

If a future phase added a genuinely justified weighting — one derived from measured
per-operation cost on specified hardware — the scalar could become the headline result. But that
would be a *timing*-derived weighting, and it would carry every confound in
[timing measurement](/design/timing-measurement.md) while being presented as the portable
quantity. That is a strong argument against revisiting this decision in that direction.

## Related

* [Step-counting semantics](/design/step-counting-semantics.md) — the vector's components.
* [FR-5](/requirements/functional-requirements.md) — the requirement.
* [US-4](/requirements/user-stories.md) — knowing when a number is meaningless.