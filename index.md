---
okf_version: "0.2"
---

# Turbo Enigma Knowledge Base

A [knowledge bundle](/README.md) documenting **the Algorithm Run Service** — an HTTP API
that runs a submitted array of integers through a set of registered sorting algorithms and
returns a machine-readable comparison of their results, step counts, and timings.

This repository contains **documentation only**. There is no service source code here yet;
see [Phase 1 scope](/requirements/scope-and-phasing.md) for what the documented system must
do and [the roadmap](/roadmap/index.md) for the build order.

## Start here

New to this bundle, read in this order:

1. [Product brief](/requirements/product-brief.md) — what this is and who it is for.
2. [Functional requirements](/requirements/functional-requirements.md) — what it must do.
3. [System context](/architecture/system-context.md) — the shape of the thing.
4. [Step-counting semantics](/design/step-counting-semantics.md) — the single most
   consequential design decision in the system.

## Product

* [Product brief](/requirements/product-brief.md)
* [Functional requirements](/requirements/functional-requirements.md)
* [User stories](/requirements/user-stories.md)
* [Scope and phasing](/requirements/scope-and-phasing.md)

## Architecture

* [System context](/architecture/system-context.md) — actors, boundaries, external systems.
* [Component view](/architecture/component-view.md) — modules and their contracts.
* [Request lifecycle](/architecture/request-lifecycle.md) — what happens on `POST /v1/runs`.
* [Data model](/architecture/data-model.md) — entities and their lifecycles.

## Design

* [Run execution model](/design/run-execution-model.md) — synchronous vs. queued.
* [Step-counting semantics](/design/step-counting-semantics.md) — what a "step" means.
* [Timing measurement](/design/timing-measurement.md) — why wall-clock is not a ranking.
* [Algorithm registry](/design/algorithm-registry.md) — the extension contract.
* [Input validation](/design/input-validation.md) — limits and rejection rules.

## API

* [Run comparison endpoint](/api/run-comparison-endpoint.md) — `POST /v1/runs`.
* [Run retrieval endpoint](/api/run-retrieval-endpoint.md) — `GET /v1/runs/{run_id}`.
* [Error model](/api/error-model.md) — status codes and error body shape.
* [Worked example](/api/worked-example.md) — a real request with measured results.

## Non-functional requirements

* [NFR overview](/nfr/nfr-overview.md) — index and how targets are stated.
* [Performance](/nfr/performance.md)
* [Scalability](/nfr/scalability.md)
* [Reliability](/nfr/reliability.md)
* [Security](/nfr/security.md)
* [Observability](/nfr/observability.md)
* [Maintainability](/nfr/maintainability.md)

## Algorithm reference

* [Complexity reference](/algorithms/complexity-reference.md) — the comparison table.
* [Bubble sort](/algorithms/bubble-sort.md)
* [Merge sort](/algorithms/merge-sort.md)
* [Quicksort](/algorithms/quicksort.md)

## Decisions

* [ADR-001: metric vector, not a scalar step count](/decisions/adr-001-metric-vector-over-scalar.md)
* [ADR-002: synchronous execution in phase 1](/decisions/adr-002-synchronous-execution.md)
* [ADR-003: median-of-three pivot policy](/decisions/adr-003-pivot-policy.md)
* [ADR-004: OKF as the bundle format](/decisions/adr-004-okf-as-bundle-format.md)

## Roadmap and risk

* [Phase 1: sort comparison](/roadmap/phase-1-sort-comparison.md)
* [Future phases](/roadmap/future-phases.md)
* [Risk register](/risks/risk-register.md)

## Reference

* [Glossary](/glossary.md) — domain terms, especially the precise meaning of "step".
* [Measurement provenance](/references/measurement-provenance.md) — how the numbers in
  these docs were produced and what they are not.
