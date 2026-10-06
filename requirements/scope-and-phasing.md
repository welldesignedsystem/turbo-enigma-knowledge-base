---
type: Requirements Document
title: Scope and phasing
description: What phase 1 includes, what it explicitly excludes, and the criteria for calling it done.
tags: [requirements, scope, phasing]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Scope and phasing

## Phase 1 — Sort comparison

One synchronous endpoint, three algorithms, deterministic step counts, verified output.

**In scope:**

* `POST /v1/runs` — run the selected algorithms over an integer array and return results
  synchronously ([FR-1 … FR-8](/requirements/functional-requirements.md)).
* `GET /v1/runs/{run_id}` — retrieve a prior run's stored result.
* `GET /v1/algorithms` — publish the registry and its properties (FR-9).
* `GET /healthz`, `GET /readyz`.
* The three algorithms of [phase 1](/roadmap/phase-1-sort-comparison.md), each conforming to
  the [registry contract](/design/algorithm-registry.md).
* Instrumentation implementing [step-counting semantics](/design/step-counting-semantics.md)
  exactly, including swap-is-2-moves.
* Reference-sort output verification (FR-6), failing the run on mismatch.
* `options.iterations` and `options.warmup_iterations` (FR-10).
* Durable storage of run results, sufficient for retrieval (US-7).

**Explicitly out of phase 1:**

| Deferred | Why | Phase |
|---|---|---|
| Asynchronous / queued execution | Synchronous is sufficient for the input sizes this service accepts, and it removes a distributed-systems problem from the first deliverable ([ADR-002](/decisions/adr-002-synchronous-execution.md)) | 2 |
| `GET /v1/runs` listing and filtering | Phase 1 stores runs for retrieval by ID; browsing them is not needed to teach anything | 2 |
| Heapsort, insertion sort, counting sort | The trio already spans the contrast space; more algorithms add breadth, not insight | 2 |
| 3-way partition / duplicate-key fix for quicksort | Documented as a known limitation on purpose ([ADR-003](/decisions/adr-003-pivot-policy.md), US-3) | 2 |
| Full step-by-step trace output | Large, and useful for a debugger rather than an API | 3 |
| Authentication | Phase 1 has no data worth protecting and no auth story written ([security NFR](/nfr/security.md)) | 2 |
| Rate limiting per caller | Needs an identity to rate-limit against | 2 |
| Multiple input types (floats, strings) | Requires re-specifying FR-1's numeric contract and raises the `<=` vs `<` comparison question for equal-but-distinct keys | 3 |
| Persisted historical trending | Needs a defensible methodology first; trending noisy timings is worse than not trending them | 3 |

**Out of scope permanently:** accepting user-submitted algorithm code, and any composite
single-number "performance score". See
[the product brief's exclusions](/requirements/product-brief.md#what-it-explicitly-is-not).

## Phase 1 definition of done

The phase is done when all of these are true. Each is checkable; none is currently true,
because the service does not exist.

1. All of FR-1 … FR-12 are implemented, or a deviation is documented in an ADR.
2. `comparisons` and `moves` match the
   [worked example's golden numbers](/api/worked-example.md) for
   `[5,3,8,1,9,2]`: bubble `15`/`16`, merge `10`/`32`, quicksort `8`/`30`. These are
   regression fixtures, and a mismatch means the counting semantics drifted.
3. A regression test proves the [FR-4 defensive-copy](/design/run-execution-model.md#decision-1--one-defensive-copy-per-execution)
   requirement: running all three algorithms produces the same numbers as running each in
   isolation.
4. An injected-fault test proves FR-6: a deliberately broken algorithm fails the run and
   never returns a successful response containing wrong data.
5. Quicksort's worst case is demonstrated in a test: n = 32 all-duplicate input yields
   589 comparisons, matching [the documented figure](/algorithms/quicksort.md#the-pivot-trap).
6. The timing warning required by FR-7 appears in the response body and in the API docs.
7. The [NFR targets](/nfr/nfr-overview.md) are measured against a real build, and this
   bundle's NFRs move from "target" to "measured" with provenance recorded.
8. Every [ADR](/decisions/index.md) is either still accurate or superseded by a new one.

## Why phase 1 is synchronous

The phase 1 input limit (10,000 elements) bounds [bubble sort](/algorithms/bubble-sort.md)
to 49,995,000 comparisons, which completes in well under a second in a compiled language.
An asynchronous model therefore buys nothing here while costing a job queue, worker
lifecycle, retry semantics, and a second API surface (`POST` returning `202` plus polling)
before a single result has been shown to be useful.

The synchronous choice is safe to migrate from because the response shape already carries
`run_id`, `created_at`, and `execution_mode`, and runs are already persisted — so the
transition is to return `202` plus a status URL and keep the same result document. That path
is sketched in [future phases](/roadmap/future-phases.md). See
[ADR-002](/decisions/adr-002-synchronous-execution.md).

## The thing most likely to be wrong

If phase 1 is going to disappoint, it will be because [bubble sort](/algorithms/bubble-sort.md)
dominates every run and makes the service feel slow and the comparison feel obvious. That is
the correct trade — bubble sort is in the trio because its quadratic behaviour is the lesson,
and the alternative is a service whose runs feel uniformly fast and teach nothing. The
mitigation is the [input limit](/design/input-validation.md) plus honest timing labels, not
removing bubble sort.