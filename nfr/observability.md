---
type: Non-Functional Requirement
title: Observability
description: Logging, metrics, and correlation requirements, including what must not be instrumented.
tags: [nfr, observability, logging, metrics]
status: draft
nfr_state: target
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Observability

One requirement here is binding and unusual ([NFR-O3](#nfr-o3--step-counts-not-logged-as-performance-telemetry)).
The rest are targets.

## NFR-O1 — Every request correlatable

| Requirement | Statement |
|---|---|
| NFR-O1.1 | Every response, success and failure, **MUST** carry a `request_id`. |
| NFR-O1.2 | `request_id` **MUST** appear in every server-side log line for that request. |
| NFR-O1.3 | A `run_id` **MUST** be attached to the log context as soon as it exists. |

`request_id` is the correlation handle named in
[the error model](/api/error-model.md) and is what makes NFR-X3.3 satisfiable: the detail a
caller must not receive has to be *findable* by an operator.

`run_id` joins the context at
[step 3 of the lifecycle](/architecture/request-lifecycle.md#3-run-orchestrator), before any
algorithm runs — so a verification failure at step 4 is traceable to the run that produced it.

## NFR-O2 — Per-algorithm observability

| Requirement | Statement |
|---|---|
| NFR-O2.1 | Emitted per run: `input_length`, `algorithms` selected, per-algorithm `comparisons`, `moves`, `elapsed_ns`, and the [step-counting version](/design/step-counting-semantics.md). |
| NFR-O2.2 | Emitted per run: `iterations`, `warmup_iterations`, and `execution_mode`. |
| NFR-O2.3 | Emitted per run: outcome — `succeeded`, `verification_failed` (naming the algorithm), or `budget_exceeded`. |
| NFR-O2.4 | Emitted per run: `input_digest`. |

`comparisons` and `moves` are safe and valuable to record, because they are deterministic and
portable. Their distributions across runs are genuinely informative: an unexpected spike at a
given `input_length` means clients are sending a different *shape* of input than usual, which
is exactly the kind of change
[step counts detect and timings do not](/algorithms/complexity-reference.md#degenerate-inputs).

`input_digest` supports deduplicating repeated identical submissions during analysis without
logging the input itself.

### Logs must not contain the input

[NFR-X3.2](/nfr/security.md#nfr-x3--errors-leak-no-internals-or-input-data) applies to logs with
at least the force it applies to response bodies. A log line containing a 10,000-element array is
a large, expensive record that leaks caller data into log storage.

Record `input_length` and `input_digest`. If the values themselves are ever needed, they are
recoverable from the stored run by `run_id` — which is the right place for them.

## NFR-O3 — Step counts are not performance telemetry

**This requirement is binding, and it is the unusual one.**

| Requirement | Statement |
|---|---|
| NFR-O3.1 | `elapsed_ns` **MUST NOT** be aggregated into a dashboard, alert, or SLO that represents the service's performance. |
| NFR-O3.2 | No metric may present "algorithm X is faster than algorithm Y" derived from `elapsed_ns`. |
| NFR-O3.3 | Any dashboard that displays `elapsed_ns` **MUST** display the [environment-dependent disclaimer](/design/timing-measurement.md) alongside it. |

The reasoning: a chart of `elapsed_ns` becomes a performance claim within a week of being built,
because that is what dashboards are for. Once someone has watched bubble sort's `elapsed_ns`
diverge over a month they will conclude it is getting slower — and it may be, or the machine may
have been throttling, or the input mix may have changed. The methodology
[does not support the conclusion](/design/timing-measurement.md#why-it-cannot-rank-algorithms),
but the chart will have made it anyway.

Step counts are the metrics to track, precisely because they are stable. If they move, the
inputs or the implementations moved, and that is a real finding.

### Alert on things that are actually defects

| Alert | Why it matters |
|---|---|
| Any `verification_failed` | Always a defect ([NFR-R1](/nfr/reliability.md#nfr-r1--no-incorrect-output-ever)). Never expected, never ignorable. |
| `verification_failed` rate spike | Suggests either a regression or an attempt to find one. |
| `run_budget_exceeded` rate | Clients sending inputs the limits do not anticipate. |
| Any `internal_error` | Should be zero. |
| Registry validation failure at startup | Prevents boot ([NFR-R3](/nfr/reliability.md#nfr-r3--registry-validated-at-startup)). |

## Health endpoints

| Endpoint | Checks | Used by |
|---|---|---|
| `GET /healthz` | Process liveness only. | Liveness probe. |
| `GET /readyz` | Result store reachability. | Readiness probe. |

`healthz` deliberately does **not** check the store. A liveness probe that fails during a
dependency outage triggers a restart loop, converting a degradation into an outage — the
dependency comes back, but the instance has been killed repeatedly in the meantime and may not
recover before the next probe. Dependency health belongs in `readyz`.

## Tracing

Distributed tracing is **not** required in phase 1. There are no internal network hops
([component view](/architecture/component-view.md#dependency-direction)), so a trace would be a
single span with no children — `request_id` correlation delivers the available value. It becomes
worthwhile at phase 2, when the queue and workers introduce genuine hops.