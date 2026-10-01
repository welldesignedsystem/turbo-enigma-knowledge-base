---
type: Roadmap
title: Phase 1 — Sort comparison
description: Build order and exit criteria for the first deliverable of the Algorithm Run Service.
tags: [roadmap, phase-1, delivery]
status: draft
stale_after: 2027-01-02T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Phase 1 — Sort comparison

One synchronous endpoint, three algorithms, deterministic step counts, verified output. Scope
boundary in [scope and phasing](/requirements/scope-and-phasing.md); target exit criteria in
[the definition of done](/requirements/scope-and-phasing.md#phase-1-definition-of-done).

## Prerequisites before any code is written

Two decisions must be made first, because they change the design rather than follow from it.

**1. Choose the implementation language.**
[NFR-P1](/nfr/performance.md#what-moves-these-numbers) is technology-dependent: 50 million
interpreted operations can be two orders of magnitude slower than compiled, which decides
whether the input limit of 10,000 is achievable at all. This is not a preference to be settled
during implementation.

**2. Choose the result store.**
[FR-8](/requirements/functional-requirements.md) requires durable storage, and the
persist-before-respond ordering
([request lifecycle](/architecture/request-lifecycle.md#step-6--persist-before-responding))
depends on it. The store must support single-document writes and point lookups; nothing more is
needed, and nothing less will satisfy retrieval.

Neither choice is made here because [this is a documentation repository](/README.md).

## Build order

The order matters: instrumentation before API, API before algorithms, because the golden-number
tests are what make each step verifiable.

| Step | Work | Done when |
|---|---|---|
| 1 | Implement [step-counting semantics `v1`](/design/step-counting-semantics.md) as shared instrumentation: `comparisons`, `moves`, per-execution reset. | Counters reset correctly across repeat executions ([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)). |
| 2 | Implement the three algorithms conforming to the [registry contract](/design/algorithm-registry.md), each with its counters. | Each matches its [golden numbers](#golden-numbers). |
| 3 | Implement the **reference sort**, from a different implementation family where practical. | Independent — it is not one of the three. |
| 4 | Build the orchestrator: copy-per-execution, verification gate, sequential deterministic order. | All-algorithms-together equals each-in-isolation. |
| 5 | Build the validator implementing [input validation](/design/input-validation.md). | Every rule in the evaluation-order table has a test. |
| 6 | Build the HTTP layer and the [error envelope](/api/error-model.md). | Every documented code is reachable and correctly shaped. |
| 7 | Build the result store integration and [the endpoints](/api/index.md). | `201` then `GET` by `run_id` round-trips. |
| 8 | Add the `FR-7` timing disclaimer and `GET /v1/algorithms`. | Disclaimer present in **every** response. |
| 9 | Load-test against [NFR-P1](/nfr/performance.md) and calibrate the time budget. | Targets met, or targets revised with a record of why. |
| 10 | Measure the NFRs and update this bundle with provenance. | No NFR still reads `target` without a stated reason. |

Steps 1–4 are the service. Steps 5–8 make it usable. Steps 9–10 make its claims honest.

## Golden numbers

The regression fixtures for [NFR-M2](/nfr/maintainability.md#nfr-m2--golden-numbers-guard-against-drift).
A mismatch is a defect, never an adjustment.

| Input | Assertion | Source |
|---|---|---|
| `[5,3,8,1,9,2]` | bubble `15`/`16`, merge `10`/`32`, quick `8`/`30` | [worked example](/api/worked-example.md) |
| `[8,7,6,5,4,3,2,1]`, n = 8 | bubble `28` comparisons = `n(n−1)/2` | [step-counting semantics](/design/step-counting-semantics.md) |
| `[1,…,32]` | bubble `31` comparisons, `0` moves | [complexity reference](/algorithms/complexity-reference.md) |
| 32 copies of `7` | quick **`496`** comparisons — the [pivot trap](/algorithms/quicksort.md#the-pivot-trap) | [ADR-003](/decisions/adr-003-pivot-policy.md) |
| `[1,…,32]` | quick `103` comparisons (median-of-three) | [complexity reference](/algorithms/complexity-reference.md) |
| `[]` and `[x]` | all algorithms: `0` comparisons, `0` moves | [FR-2](/requirements/functional-requirements.md) |

All verified under `step_counting_version: "v1"`. Provenance in
[measurement provenance](/references/measurement-provenance.md).

**Quicksort assertions must be checked against instrumented output, not hand-derived
expectations.** A manual trace of quicksort does not reliably reconcile with an implementation —
documented at
[the worked example](/api/worked-example.md#the-quicksort-trace-above-is-wrong-and-that-is-the-point).

## Required non-functional tests

Each maps to a binding or target NFR. These are what make the service trustworthy rather than
merely working.

| Test | NFR | Asserts |
|---|---|---|
| Defensive copy | [FR-4](/requirements/functional-requirements.md) | Running all three together equals each in isolation. |
| Injected fault | [NFR-R1](/nfr/reliability.md#nfr-r1--no-incorrect-output-ever) | A deliberately broken algorithm fails the run; no wrong data is ever returned. |
| Counters reset | [Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution) | `iterations: 100` yields the same step counts as `iterations: 1`. |
| Buffer freshness | [FR-4](/requirements/functional-requirements.md) | `iterations: 100` does not let an in-place algorithm re-sort its own output. |
| Boundary input | [FR-2](/requirements/functional-requirements.md) | `n−1`, `n`, `n+1` behave correctly at the limit. |
| Type rejection | [FR-1](/requirements/functional-requirements.md) | Floats, booleans, strings, `null`, and out-of-`int64` values are all rejected. |
| JavaScript ceiling | [FR-1](/requirements/functional-requirements.md) | Values above 2⁵³−1 are rejected. |
| Empty algorithm list | [FR-3](/requirements/functional-requirements.md) | `algorithms: []` is `400`, not "all". |
| Unknown algorithm | [FR-3](/requirements/functional-requirements.md) | The whole request fails; no silent skipping. |
| Budget enforcement | [NFR-R2](/nfr/reliability.md#nfr-r2--per-run-time-budget-enforced) | An over-budget run aborts with `422` and returns no partial results. |
| Disclaimer present | [FR-7](/requirements/functional-requirements.md) | Every response carries it. |
| Retrieval is verbatim | [NFR-R4](/nfr/reliability.md#nfr-r4--stored-runs-are-immutable) | `GET` returns the stored document without recomputation. |
| Determinism | [US-1](/requirements/user-stories.md) | The same input twice returns identical step counts. |

## Exit criteria

The [definition of done](/requirements/scope-and-phasing.md#phase-1-definition-of-done) is
normative; the practical check is that all eight conditions hold, including that every golden
number above passes and that [every NFR](/nfr/nfr-overview.md) has either been measured or
explicitly re-labelled a target with a reason.

## What phase 1 deliberately will not have

Restated because it is the most likely thing to be added by well-meaning scope creep:

* No queue, no workers ([ADR-002](/decisions/adr-002-synchronous-execution.md)).
* No authentication ([security](/nfr/security.md#what-phase-1-does-not-provide)).
* No `GET /v1/runs` listing.
* No 3-way partition — quicksort keeps its duplicate-key cliff
  ([ADR-003](/decisions/adr-003-pivot-policy.md)).
* No timing dashboards or trends
  ([NFR-O3](/nfr/observability.md#nfr-o3--step-counts-not-logged-as-performance-telemetry)).
* No step-by-step trace output.