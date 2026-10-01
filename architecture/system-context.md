---
type: Architecture Document
title: System context
description: The Algorithm Run Service boundary, its actors, and the systems it does and does not depend on.
tags: [architecture, c4, context]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# System context

## The service in one line

The Algorithm Run Service accepts an array of integers over HTTP, runs it through a set of
in-process sorting algorithms, verifies each result, and returns a JSON comparison of
step counts and timings.

## Actors

| Actor | What it wants | Phase 1 access |
|---|---|---|
| **Student** | To see step counts for an array and understand why they differ | Unauthenticated |
| **Instructor** | To share a reproducible `run_id` and to find an input that breaks an algorithm | Unauthenticated |
| **Integrator / developer** | A stable, documented contract; to add an algorithm via the [registry](/design/algorithm-registry.md) | Unauthenticated |
| **Operator** | Health signals, logs, and safe deployment | Deployment-time |

No actor is authenticated in phase 1. That is a deliberate phase-1 simplification with a
known cost, recorded in [the security NFR](/nfr/security.md) and
[the risk register](/risks/risk-register.md).

## Boundary

```
                    ┌──────────────────────────────────────────┐
   Student  ───────▶│                                          │
   Instructor ─────▶│        Algorithm Run Service             │
   Integrator ─────▶│                                          │
                    │  ┌────────────────────────────────────┐  │
                    │  │ HTTP layer      (routing, JSON)   │  │
                    │  │ Validation      (FR-1, FR-2)      │  │
                    │  │ Run orchestrator(FR-4, FR-6)      │  │
                    │  │ Algorithm registry + instrumented  │  │
                    │  │                 algorithms (FR-5) │  │
                    │  │ Result store    (FR-8)            │  │
                    │  └────────────────────────────────────┘  │
                    └────────────────┬─────────────────────────┘
                                     │
                              ┌──────▼───────┐
                              │  Durable     │
                              │  result store│
                              └──────────────┘
```

## External dependencies, phase 1

| Dependency | Required? | Notes |
|---|---|---|
| Durable store for run results | **Yes** | Needed for [FR-8](/requirements/functional-requirements.md) and US-7 retrieval. Small: one document per run. |
| Job queue | **No** | Deliberately absent. See [ADR-002](/decisions/adr-002-synchronous-execution.md). |
| Identity provider | **No** | Deferred. See [security](/nfr/security.md). |
| External reference-sort service | **No** | The reference sort is computed in-process. Calling out to another service would add a network failure mode to the one path that must never be wrong ([FR-6](/requirements/functional-requirements.md)). |
| Clock | Yes | Monotonic for `elapsed_ns`, wall-clock for `created_at`. See [timing measurement](/design/timing-measurement.md). |
| Randomness | No | Phase 1 is fully deterministic given an input. |

The absence of a queue and an identity provider is the whole shape of phase 1. Everything
in [the component view](/architecture/component-view.md) exists to serve one request in one
process.

## Trust boundary

There is exactly one trust boundary: **the HTTP request**. Everything past validation is
first-party code operating on first-party data.

The reason this matters is that the obvious way to extend this service — letting callers
submit an algorithm — would place user-supplied code *inside* the boundary. That is a
remote code execution surface, and it is why algorithms are compiled into the binary via
the [registry](/design/algorithm-registry.md) instead of being uploaded. See
[the product brief](/requirements/product-brief.md#what-it-explicitly-is-not).

## Environmental dependencies that shape the architecture

* **The algorithms run in-process, on the request goroutine/thread.** There is no
  sandbox because there is no untrusted code. CPU cost is therefore paid by the API
  process, which is why [FR-2's input limit](/design/input-validation.md) is a correctness
  feature and not only a politeness one: an unbounded array is a denial-of-service vector
  even with no malicious intent.
* **Timing measurements are taken inside the same process that is serving other requests.**
  This makes `elapsed_ns` noisy by construction. The architecture does not try to fix that;
  it documents it and instructs callers not to rank on it ([FR-7](/requirements/functional-requirements.md)).
  Isolating measurements into dedicated workers is a phase-3 idea, and even then the
  numbers remain machine-specific.
* **Step counts do not depend on any of this.** They are functions of input and algorithm
  alone, which is what makes the service's central promise portable across environments.

## Deployment shape

A single stateless process plus its result store. No internal network hops between
components, because phase 1 has no components that need to be separately deployed.

Statelessness is what allows horizontal scaling later: because `run_id` is generated by the
service and results are written to the shared store, any instance can serve any subsequent
`GET /v1/runs/{run_id}`.

## C4 navigation

* Zoom in: [component view](/architecture/component-view.md)
* Follow a request: [request lifecycle](/architecture/request-lifecycle.md)
* Understand the data: [data model](/architecture/data-model.md)