---
type: Decision Record
title: "ADR-002: Synchronous execution in phase 1"
description: Runs execute to completion on the request thread and return results inline. The async migration path is documented.
tags: [adr, execution, architecture, phasing]
status: accepted
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# ADR-002: Synchronous execution in phase 1

**Status:** accepted · **Date:** 2026-10-02 · **Deciders:** bundle author (agent-drafted,
unreviewed — see [trust and review](/README.md#trust-and-review))

## Context

The service is described as performing "test runs". The word suggests queued, asynchronous
execution — submit a job, get an identifier, poll for results. That is the default architecture
for long-running work, and most API references for benchmarking or analysis tools use it.

Phase 1 also imposes a hard input limit of 10,000 elements
([input validation](/design/input-validation.md)), which bounds the slowest algorithm's work.

## Decision

Phase 1 executes runs **synchronously**: on the request-handling thread, to completion, with
results returned in the response body. No queue, no worker pool, no job state, no polling.

`execution_mode` is reported as `"synchronous"` in every response
([FR-8](/requirements/functional-requirements.md)).

## Rationale

**The input limit makes long-running work impossible.** At n = 10,000,
[bubble sort](/algorithms/bubble-sort.md) performs 49,995,000 comparisons
([verified](/design/step-counting-semantics.md#consequence-2--bubble-sorts-worst-case-is-exactly-nn12))
— comfortably within a second in a compiled language and inside a
[2-second budget](/nfr/performance.md#nfr-p1--worst-case-run-latency-within-budget). An
asynchronous architecture is for work whose duration the caller cannot wait for. Phase 1's
duration is bounded and short.

**Asynchrony would delay the first result, not shorten it.** A queue adds enqueue latency,
worker dispatch, and a polling round-trip *before* the same computation. For a workload measured
in hundreds of milliseconds, that is a net regression in time-to-answer.

**It removes a distributed-systems problem from the first deliverable.** An asynchronous design
requires: a queue, worker lifecycle management, retry and idempotency semantics, job status
transitions, partial-failure handling, and a second API surface for polling. None of that teaches
anything about sorting algorithms, and all of it is where the bugs would be. The service's value
is its measurement semantics ([the product brief](/requirements/product-brief.md)); spending phase
1 on job orchestration spends it on the wrong problem.

**It keeps the failure model simple.** Exactly four failure points, one per requirement
([request lifecycle](/architecture/request-lifecycle.md#the-four-failure-points)). There is no
"job queued" state in which a run can be lost, and no partial result to reason about.

## The cost, stated honestly

Synchronous execution means a long run **holds a connection and a thread** for its duration.
Under sustained load this is exactly the wrong shape: no request can be cancelled once
submitted, and one slow run occupies a worker. This is accepted for phase 1 because phase 1
bounds the duration — and it is the reason the
[per-run time budget](/nfr/reliability.md#nfr-r2--per-run-time-budget-enforced) exists as a
hard requirement rather than an optimisation.

## Migration path

The design deliberately keeps the transition cheap:

1. `run_id`, `created_at`, and `execution_mode` are **already** in the response
   ([data model](/architecture/data-model.md#run)).
2. Runs are **already persisted before the response is sent**, so a result document already
   exists independently of the HTTP response.
3. The result document shape does not change; only the status code and the arrival of the
   document do.

The phase-2 migration is therefore:

| Phase 1 | Phase 2 |
|---|---|
| `POST /v1/runs` → `201` + full result | `POST /v1/runs` → `202` + `{run_id, status_url}` |
| Results inline in the `POST` response | Results via `GET /v1/runs/{run_id}` |
| `execution_mode: "synchronous"` | `execution_mode: "asynchronous"` |
| Cancellation impossible | `DELETE /v1/runs/{run_id}` |

The result document already exists and already carries `execution_mode`, so **stored phase-1
runs remain interpretable after migration** — which is the point of recording it. A consumer
that reads `execution_mode` needs no other change to handle both.

## When this decision should be revisited

Revisit if **any** of these becomes true:

* The [input limit](/design/input-validation.md) is raised, making legitimate runs long enough
  that a caller should not have to wait.
* Sustained load makes thread occupancy the bottleneck rather than CPU
  ([NFR-S1](/nfr/scalability.md#nfr-s1--concurrency-bounded-by-cpu-work-not-by-memory)).
* A future workload — larger inputs, or
  [step-by-step traces](/roadmap/future-phases.md) — can take seconds to minutes.

Do **not** revisit it merely because asynchronous is the more common pattern for this shape of
problem. The input limit is what makes this decision right, and the input limit is a deliberate
choice ([the input limit exists to protect bubble sort](/design/input-validation.md#why-the-length-limit-exists)).

## Related

* [Run execution model](/design/run-execution-model.md) — the mechanism.
* [Scope and phasing](/requirements/scope-and-phasing.md) — what the deferral buys.
* [Future phases](/roadmap/future-phases.md) — the phase-2 scope.