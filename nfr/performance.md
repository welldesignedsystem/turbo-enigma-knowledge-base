---
type: Non-Functional Requirement
title: Performance
description: Latency, verification overhead, and timing methodology requirements. All values are unvalidated targets.
tags: [nfr, performance, latency]
status: draft
nfr_state: target
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Performance

**Every figure on this page is an unvalidated target.** The service does not exist
([scope and phasing](/requirements/scope-and-phasing.md)). Values are technology-neutral
because no implementation language has been chosen, and they will need revision once one has —
see [what moves these numbers](#what-moves-these-numbers).

## NFR-P1 — Worst-case run latency within budget

| Input size | Target end-to-end p99 |
|---|---|
| `n ≤ 100` | < 50 ms |
| `n ≤ 1,000` | < 250 ms |
| `n ≤ 10,000` (the [limit](/design/input-validation.md)) | < 2 s |

The n = 10,000 row is the binding one, because that is where
[bubble sort](/algorithms/bubble-sort.md) performs 49,995,000 comparisons
([verified](/design/step-counting-semantics.md#consequence-2--bubble-sorts-worst-case-is-exactly-nn12)).
The target is dominated entirely by one algorithm's quadratic cost on one input size.

**The request body size limit does not imply a latency limit.** An array of 10,000 integers is
small on the wire and expensive to sort, which is why
[NFR-X4](/nfr/security.md#nfr-x4--resource-exhaustion-bounded-per-request) and NFR-P1 are
coupled: the input limit *is* the latency control.

### What moves these numbers

A **compiled** implementation should meet these targets with headroom. An **interpreted** one
may not: roughly 50 million interpreted operations is commonly two orders of magnitude slower
than the same logic compiled. If an interpreted language is chosen, either the input limit drops
by a similar factor or NFR-P1's third row is missed. This is the single largest open risk in the
performance section, and it is a decision to make **before** implementation, not after.

## NFR-P2 — Verification overhead bounded

The [reference sort](/glossary.md#reference-sort) and the per-algorithm output comparison
([FR-6](/requirements/functional-requirements.md)) add a fixed cost: one additional sort of the
input plus n comparisons per algorithm.

| Requirement | Target |
|---|---|
| Verification as a fraction of total run time | ≤ 25% |
| Verification adds no more than | ~1 additional sort's worth of comparisons |

The 25% figure holds because bubble sort dominates the run, so a merge sort used as reference is
negligible against it. **At small `n` the fraction will be much higher** — for `n = 6`,
verification is most of the work — and the target is explicitly a *worst-case* claim, not an
average one. A monitoring alert on this ratio should be evaluated only on large inputs.

Verification is not optional and is not negotiable under load. NFR-R5 covers why.

## NFR-P3 — Timing measurement methodology bounded

Requirements on how `elapsed_ns` is produced, from
[timing measurement](/design/timing-measurement.md):

| Requirement | Statement |
|---|---|
| NFR-P3.1 | Clock **MUST** be monotonic. A wall clock that can jump backwards would produce negative durations. |
| NFR-P3.2 | Measurement window **MUST** exclude parsing, validation, serialisation, and the verification pass. |
| NFR-P3.3 | Reported value **MUST** be the median of `iterations` samples ([FR-10](/requirements/functional-requirements.md)). |
| NFR-P3.4 | Warm-up iterations **MUST** be discarded entirely. |
| NFR-P3.5 | `elapsed_ns` **MUST NOT** be used to rank algorithms, and every response **MUST** carry the [disclaimer](/api/run-comparison-endpoint.md#response-fields) ([FR-7](/requirements/functional-requirements.md)). |
| NFR-P3.6 | `elapsed_ns` **MUST NOT** be persisted as a basis for historical trending of algorithm performance. |

NFR-P3.6 is the one most likely to be violated by accident, because storing timings and charting
them is the obvious next thing to build. It is prohibited because trending them would publish a
performance claim the methodology
[does not support](/design/timing-measurement.md#why-it-cannot-rank-algorithms) — and would do
so with the authority of the service's own storage.

## Why there is no throughput target

Throughput depends on request arrival rate, which nobody has stated
([NFR overview](/nfr/nfr-overview.md#what-is-missing-and-honestly-so)). A throughput number here
would be invented.

What *can* be said now: each run consumes CPU proportional to bubble sort's cost, so the service
is **CPU-bound**, and throughput scales with cores and not with instance memory or request
queue depth. See [scalability](/nfr/scalability.md).

## Latency by input shape, not just by size

Two requests of the same length can differ by more than an order of magnitude in run time,
because step counts depend on input shape
([the complexity reference](/algorithms/complexity-reference.md#degenerate-inputs)). At n = 32
quicksort performs between 103 and 496 comparisons depending on input.

**Any percentile computed across a mixed workload will be dominated by whatever mix of input
shapes arrived,** not by the algorithm. NFR-P1's rows are therefore specified against a stated
worst-case input, and any SLO derived from them must state which input shape it assumed.

## Interactions with other NFRs

* [NFR-S1](/nfr/scalability.md#nfr-s1--concurrency-bounded-by-cpu-work) — CPU-bound work does not
  parallelise by adding concurrency; it parallelises by adding cores.
* [NFR-R2](/nfr/reliability.md#nfr-r2--per-run-time-budget-enforced) — the time budget is the
  runtime enforcement of NFR-P1, for inputs that are valid but unexpectedly expensive.
* [NFR-X4](/nfr/security.md#nfr-x4--resource-exhaustion-bounded-per-request) — the input limit is
  the preventive control; the time budget is the detective one.