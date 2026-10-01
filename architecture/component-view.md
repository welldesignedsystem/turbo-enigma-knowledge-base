---
type: Architecture Document
title: Component view
description: The internal modules of the Algorithm Run Service, their responsibilities, and the contracts between them.
tags: [architecture, c4, components]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Component view

Six modules inside one process. The boundaries are logical, not deployment boundaries —
phase 1 deploys them together, but keeping them separable is what makes phase 2 (a job
queue) a wiring change rather than a rewrite.

## Components

### 1. HTTP layer

Routing, method/path dispatch, content negotiation, request size caps, and serialisation of
responses and [errors](/api/error-model.md).

**Knows about:** HTTP, JSON, status codes.
**Does not know about:** algorithms, metrics, what a run is.

Deliberately thin. Every interesting failure is a domain failure expressed as an error
value, not an HTTP concern.

### 2. Validator

Enforces [FR-1](/requirements/functional-requirements.md) (integer array, `int64` range,
JavaScript precision ceiling), [FR-2](/requirements/functional-requirements.md) (length
limit), and [FR-3](/requirements/functional-requirements.md) (algorithm ID resolution).

Rules live in [input validation](/design/input-validation.md).

**Knows about:** the input contract and the set of registered algorithm IDs.
**Does not know about:** how an algorithm works.

Rejects **everything** invalid up front, before any CPU is spent. This is the only place
allowed to look at the raw request array.

### 3. Run orchestrator

The component that owns the semantics the rest of the service exists to serve. Its
responsibilities, in order:

1. Compute the reference sort of the submitted input ([FR-6](/requirements/functional-requirements.md)).
2. For each selected algorithm, in registry order:
   a. take a **defensive copy** of the input ([FR-4](/requirements/functional-requirements.md)),
   b. execute warm-up iterations, discarding all metrics,
   c. execute timed iterations, collecting a sample per iteration,
   d. verify the output against the reference sort,
   e. reduce the sample to the reported metric vector.
3. Assemble the run document and hand it to the result store.

**Knows about:** algorithm order, copy semantics, metric reduction, verification.
**Does not know about:** HTTP, storage details, or how any algorithm is implemented.

This is the component that must never be "optimised" by skipping the copy or the
verification. Both exist because the alternative is silently wrong output.

### 4. Algorithm registry

Holds the ordered set of compiled-in algorithm implementations and their declared
properties (`stability`, `in_place`, `complexity`, `pivot_policy`, `step_counting_version`).

The registry is the **single source of the algorithm list** used by three things: default
algorithm selection (FR-3), `GET /v1/algorithms` (FR-9), and execution order (FR-4).
Deriving those from one table is what stops them drifting apart.

Contract: [algorithm registry](/design/algorithm-registry.md).

### 5. Instrumented algorithms

Each registered algorithm wraps a plain sorting implementation with counters. The counting
rules are normative and shared: [step-counting semantics](/design/step-counting-semantics.md).

Every implementation must:

* operate only on the buffer it is given (in-place implementations included),
* return the buffer it was given, never a substituted one,
* reset counters at the start of each execution, so warm-up and iterations cannot leak into
  one another,
* report `auxiliary_slots` as peak auxiliary usage, `0` for in-place implementations.

The "reset counters per execution" rule is the subtle one. Counters that accumulate across
iterations will produce a `comparisons` value scaled by `iterations` and break
[FR-10](/requirements/functional-requirements.md)'s guarantee that step counts come from a
single deterministic iteration.

### 6. Result store

Persists the run document so `GET /v1/runs/{run_id}` works after the response returns
([US-7](/requirements/user-stories.md)), and so two callers who submitted the same array can
confirm via `input_digest` that they did.

Write must happen **before** the HTTP response is returned, so a client that receives a
`run_id` can immediately fetch it. Entities and lifecycles: [data model](/architecture/data-model.md).

## Dependency direction

```
   HTTP layer
       │
       ▼
   Validator ──────────▶ Algorithm registry (read: ID list, properties)
       │
       ▼
   Run orchestrator ───▶ Instrumented algorithms
       │      └────────▶ Algorithm registry (read: order, properties)
       ▼
   Result store
```

Two rules, both load-bearing:

* **The validator is the only component that touches raw input.** Everything downstream
  receives already-validated, type-correct data. This is what keeps validation from being
  forgotten on the retrieval path.
* **No component may call the HTTP layer.** Errors flow outward as values, never as
  responses. A domain component returning an HTTP status code is the usual reason a
  synchronous design cannot later become queued without work.

## Where each requirement lives

| Requirement | Component |
|---|---|
| FR-1 integer contract | Validator |
| FR-2 length limit | Validator |
| FR-3 algorithm selection | Validator + registry |
| FR-4 defensive copy | Run orchestrator |
| FR-5 metric vector | Instrumented algorithms + orchestrator |
| FR-6 output verification | Run orchestrator |
| FR-7 timing labelling | HTTP layer (response assembly) |
| FR-8 run metadata | Orchestrator + result store |
| FR-9 registry endpoint | Registry + HTTP layer |
| FR-10 iterations and warm-up | Orchestrator |
| FR-11 time budget | Orchestrator |
| FR-12 verification integrity | Orchestrator |