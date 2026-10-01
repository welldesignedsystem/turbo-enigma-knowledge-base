---
type: Risk
title: Risk register
description: Known risks to the Algorithm Run Service with mitigations, warning signals, and owners.
tags: [risk, register, planning]
status: draft
stale_after: 2027-01-02T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Risk register

Risks to the service described by this bundle. Scoring is **Likelihood × Impact**, both
`low`/`medium`/`high`, assessed without code and therefore without measurement — treat the
scores as a starting argument rather than a result.

## R-01 — Silent wrong results from a stale shared buffer

| | |
|---|---|
| **Likelihood** | High — it is the easiest defect to write. |
| **Impact** | High — produces plausible, entirely wrong numbers, which is the worst failure this service has. |
| **Score** | **High** |

Bubble sort and quicksort sort in place. Sharing one buffer across algorithms makes the second
algorithm measure an already-sorted array and report best-case figures for non-best-case input
([traced in full](/design/run-execution-model.md#why-the-single-copy-per-algorithm-version-is-not-enough)).

The failure is nasty because **the numbers still look right**. In the reference trace, merge
sort's shared-buffer comparison count (10) coincidentally equals its correct average-case count,
so nothing appears broken.

**Mitigation:** a defensive copy per execution, per algorithm, per iteration
([FR-4](/requirements/functional-requirements.md)); the M2.4 test
([phase 1](/roadmap/phase-1-sort-comparison.md#required-non-functional-tests)) asserting
all-together equals each-in-isolation.

**Warning signal:** step counts that improve when `iterations` increases, or that differ between
`iterations: 1` and `iterations: 2`.

## R-02 — Step-counting semantics drift silently

| | |
|---|---|
| **Likelihood** | Medium — counting rules are easy to reinterpret during a refactor. |
| **Impact** | High — every published figure becomes wrong, with no error and no failing test. |
| **Score** | **High** |

Omitting [pivot-selection comparisons](/design/step-counting-semantics.md#rule-1--comparisons)
from quicksort is the concrete example: it yields a number matching no other implementation,
and it is entirely invisible without a golden-number test.

**Mitigation:** [step-counting semantics `v1`](/design/step-counting-semantics.md) as one
normative document rather than per-algorithm conventions; `step_counting_version` on every
response and every stored run; golden numbers asserted in tests
([NFR-M2](/nfr/maintainability.md#nfr-m2--golden-numbers-guard-against-drift)).

**Warning signal:** any golden-number test needing an "expected value" comment explaining why
it changed.

## R-03 — Interpreted-language performance misses the input limit

| | |
|---|---|
| **Likelihood** | Medium — depends entirely on the undecided language choice. |
| **Impact** | Medium — forces the input limit down or [NFR-P1](/nfr/performance.md) to be missed. |
| **Score** | **Medium** |

50 million comparisons ([bubble sort at n = 10,000](/design/step-counting-semantics.md#consequence-2--bubble-sorts-worst-case-is-exactly-nn12))
is comfortable compiled and potentially two orders of magnitude slower interpreted. The limit
was chosen on the assumption of a compiled implementation.

**Mitigation:** decide the language **before** implementation
([phase 1 prerequisites](/roadmap/phase-1-sort-comparison.md#prerequisites-before-any-code-is-written));
load-test at step 9 and either confirm or revise the limit with a recorded reason.

**Warning signal:** the target p99 for n = 10,000 missed on the first load test.

## R-04 — Quicksort's duplicate-key cliff is hit in production

| | |
|---|---|
| **Likelihood** | Medium — trivially reachable by any client. |
| **Impact** | Low–medium — a slow run, not a wrong result; the time budget catches it. |
| **Score** | **Medium** |

At n = 32, all-duplicate input costs quicksort 496 comparisons and 1,116 moves
([measured](/algorithms/quicksort.md#the-pivot-trap)). At the input limit it is roughly 50
million comparisons — bubble sort's cost, on an algorithm whose advertised worst case is
O(n log n).

**Mitigation:** deliberate, documented, and published in `known_limitations`
([ADR-003](/decisions/adr-003-pivot-policy.md)); the time budget
([NFR-R2](/nfr/reliability.md#nfr-r2--per-run-time-budget-enforced)) catches the large case.
The 3-way partition is a
[phase-2 item](/roadmap/future-phases.md#3-way-partition-for-quicksort).

**Accepted, not solved.** The cliff is also the service's most instructive behaviour
([US-3](/requirements/user-stories.md)), so fixing it is a trade, not a pure win.

## R-05 — No authentication

| | |
|---|---|
| **Likelihood** | High — it is the phase-1 state, not a defect. |
| **Impact** | Medium — anonymous compute consumption and no per-caller rate limiting. |
| **Score** | **Medium** |

Anyone reaching the service can run runs and read any `run_id` they can guess. Rate limiting
cannot be per-caller without an identity
([NFR-S1](/nfr/scalability.md#nfr-s1--concurrency-bounded-by-cpu-work-not-by-memory)).

The under-appreciated part: **run inputs are persisted**
([data model](/architecture/data-model.md#on-storing-input)), so `run_id` opacity is the only
thing protecting caller data. Acceptable for a teaching tool with no sensitive use; not
acceptable for a public deployment.

**Mitigation:** edge rate limiting and TLS before public exposure; authentication in
[phase 2](/roadmap/future-phases.md#authentication-and-rate-limiting).

**Warning signal:** a spike in `run_budget_exceeded` or request volume from unexpected sources.

## R-06 — Timing numbers are quoted as a performance claim

| | |
|---|---|
| **Likelihood** | High — it is the default behaviour of anyone who reads the API docs. |
| **Impact** | Medium — reputational for the service, misleading for the reader. |
| **Score** | **Medium** |

`elapsed_ns` is in the response because callers asked for it
([FR-7](/requirements/functional-requirements.md)). Someone will quote it, a chart will be built,
and within weeks the chart will be treated as fact — with every confound in
[timing measurement](/design/timing-measurement.md) hidden behind it.

**Mitigation:** the disclaimer in **every** response, not only the docs; NFR-O3 prohibiting
`elapsed_ns` dashboards and SLOs
([observability](/nfr/observability.md#nfr-o3--step-counts-not-logged-as-performance-telemetry));
NFR-P3.6 prohibiting persisted trends; `results[]` never metric-ordered
([ADR-001](/decisions/adr-001-metric-vector-over-scalar.md)).

**Warning signal:** a dashboard appearing with `elapsed_ns` and no disclaimer.

## R-07 — Store growth outruns the document store

| | |
|---|---|
| **Likelihood** | Medium — depends on unstated volume. |
| **Impact** | Low–medium — slow writes, eventually failed ones, which fail runs ([NFR-R5](/nfr/reliability.md#nfr-r5--degradation-is-preferred-over-wrong-answers)). |
| **Score** | **Low–medium** |

Storing `input` inline means storage grows by up to 10,000 integers per run, and there is no
listing endpoint to keep per-row operations cheap. No retention policy exists
([NFR-S3](/nfr/scalability.md#nfr-s3--store-growth-and-retention-unspecified)).

**Mitigation:** the move to object storage is already designed
([data model](/architecture/data-model.md#operationally)); retention is the open decision.

**Warning signal:** run-write latency trending up alongside store size.

## R-08 — Counter accumulation across iterations

| | |
|---|---|
| **Likelihood** | Medium — a natural implementation shortcut. |
| **Impact** | Medium — every step count scaled by `iterations`. |
| **Score** | **Medium** |

Counters that are not reset per execution report `iterations ×` the true value
([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)). This is
"only" too large by an obvious factor, which makes it easy to overlook — and it appears the
moment `iterations` is supported.

**Mitigation:** per-execution reset as a [registry contract](/design/algorithm-registry.md)
obligation; the counters-reset test asserting `iterations: 100` matches `iterations: 1`.

**Warning signal:** step counts that are an exact multiple of the correct value.

## R-09 — Every concept in this bundle is unreviewed

| | |
|---|---|
| **Likelihood** | **Certain** — it is the current state. |
| **Impact** | Medium — the design may rest on a misreading. |
| **Score** | **Medium** |

No concept carries a `verified` entry, so all are at the `unverified` trust tier
([trust and review](/README.md#trust-and-review)). The bundle is entirely agent-drafted. Its
internal consistency, link integrity, and conformance can be checked mechanically; its
*technical claims* cannot — although the measured figures are reproducible
([measurement provenance](/references/measurement-provenance.md)).

**Mitigation:** human review of the load-bearing documents first —
[step-counting semantics](/design/step-counting-semantics.md), the
[three ADRs](/decisions/index.md), and [the functional requirements](/requirements/functional-requirements.md).
Each reviewed concept gets `verified: { by: human:<id>, at: … }`, which is the only thing that
distinguishes reviewed knowledge from model output.

**This is the highest-leverage item in the register**, because everything else is engineering
work that a reviewer can check, whereas this is the gap in the basis for all of it.

## R-10 — Scope creep toward a benchmarking platform

| | |
|---|---|
| **Likelihood** | Medium — it is the natural growth direction. |
| **Impact** | Low — scope pressure, not incorrectness. |
| **Score** | **Low** |

Adding algorithms, trace output, or timing dashboards is easy to justify and each is defensible
individually. Together they turn a deterministic reference service into something whose numbers
are environment-dependent — which is the outcome
[the product brief](/requirements/product-brief.md#what-it-explicitly-is-not) rejects.

**Mitigation:** [the phase-1 "will not have" list](/roadmap/phase-1-sort-comparison.md#what-phase-1-deliberately-will-not-have);
[non-stories](/requirements/user-stories.md#non-stories) recorded as rejected rather than merely
deferred.

## Open questions for the reader

Genuinely unanswered, and worth settling before implementation:

1. **Which language?** [R-03](/risks/risk-register.md) does not close without it.
2. **What result store?**
3. **What is the intended exposure?** Local network, or public? Decides whether R-05 is a
   phase-2 item or a blocker.
4. **Is any NFR target wrong?** The numbers in [performance](/nfr/performance.md) are reasoned
   guesses. Someone who knows the intended deployment should sanity-check them.
5. **Is the measured corpus in this bundle the right teaching set?** More inputs — especially
   duplicate-heavy and adversarial — would strengthen
   [the complexity reference](/algorithms/complexity-reference.md).