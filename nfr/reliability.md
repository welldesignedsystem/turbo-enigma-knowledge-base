---
type: Non-Functional Requirement
title: Reliability
description: Correctness guarantees, time budgets, and failure behaviour. The correctness requirements are binding.
tags: [nfr, reliability, correctness, availability]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Reliability

The five requirements here are **binding**, not targets. They follow from the design and their
violation would be a correctness defect, not a missed goal. See
[the NFR overview](/nfr/nfr-overview.md#which-requirements-are-binding-versus-targets).

## NFR-R1 — No incorrect output, ever

| Requirement | Statement |
|---|---|
| NFR-R1.1 | Every algorithm's output **MUST** be verified against an independently computed reference sort before the run is reported as successful. |
| NFR-R1.2 | A verification mismatch **MUST** fail the entire run. Partial results **MUST NOT** be returned. |
| NFR-R1.3 | A `correct: false` result **MUST NOT** appear in any successful response. |

This is the service's most important reliability property and the reason it exists. Its purpose
is not to catch bugs during development — that is what tests are for — but to ensure that a bug
reaching production produces a **visible failure** rather than a plausible wrong number.

That distinction is the whole design. A silently wrong sort from this service would not merely
be bad data; it would corrupt the conclusion the caller drew, and a caller who trusts the
service has no way to detect it. That is the worst failure mode a teaching tool can have.

Implements [FR-6](/requirements/functional-requirements.md); mechanism in
[the verification gate](/design/run-execution-model.md#decision-2--the-verification-gate).

## NFR-R2 — Per-run time budget enforced

| Requirement | Statement |
|---|---|
| NFR-R2.1 | Every run **MUST** be subject to a wall-clock budget. |
| NFR-R2.2 | Exceeding it **MUST** abort the run and return `422 run_budget_exceeded` naming the budget and the algorithms that had not completed. |
| NFR-R2.3 | Partial results **MUST NOT** be returned on timeout. |

The budget is the runtime enforcement of [NFR-P1](/nfr/performance.md), covering the case the
static [input limit](/design/input-validation.md) does not predict. Its most likely trigger is
not a larger array but **a pathological shape**: a valid length with an adversarial structure.
At n = 32, quicksort performs 496 comparisons on all-duplicate input against 103 on sorted
input ([measured](/algorithms/complexity-reference.md#degenerate-inputs)).

### Why `422` and not `503`

The service is healthy; the request was merely too expensive. `503` would tell a client the
condition is transient and retryable, which is wrong — retrying the identical request would
exceed the identical budget. `422` tells it to send something smaller, which is actionable.
See [the error model](/api/error-model.md#run_budget_exceeded).

### The budget is not the input limit

Both exist because they catch different things. The input limit is preventive and O(1) to check;
the budget is detective and catches everything the limit misses. An implementation that has only
the limit has a predictable denial-of-service vector; one that has only the budget burns CPU
before rejecting.

## NFR-R3 — Registry validated at startup

| Requirement | Statement |
|---|---|
| NFR-R3.1 | The service **MUST** fail to start on any registry inconsistency: duplicate IDs, empty required fields, a partition-based algorithm with no `pivot_policy`, an unknown `step_counting_version`, or a counting-semantics mismatch. |
| NFR-R3.2 | The HTTP layer **MUST** be able to trust the registry without re-validating per request. |

Fail-fast is chosen over warn-and-continue because a registry inconsistency means the service's
own metadata about itself is wrong. `GET /v1/algorithms`
([FR-9](/requirements/functional-requirements.md)) is part of the contract: a caller that
receives a quicksort entry with no `pivot_policy` cannot interpret its worst case, and cannot
tell it is missing something.

Conditions enumerated in [the registry contract](/design/algorithm-registry.md#registry-integrity).

## NFR-R4 — Stored runs are immutable

| Requirement | Statement |
|---|---|
| NFR-R4.1 | A stored `Run` **MUST NOT** be updated or recomputed. |
| NFR-R4.2 | `GET /v1/runs/{run_id}` **MUST** return the stored document verbatim. |
| NFR-R4.3 | `Result` **MUST** carry its own copy of the behavioural flags it depended on (`in_place`, `stable`, `pivot_policy`). |

Retrieval must never recompute, or a change to an algorithm silently rewrites history: a `run_id`
from a year ago would start reporting figures from an algorithm that never produced them. R4.3
exists because the alternative — resolving those flags against the current registry — produces
exactly that corruption the moment [ADR-003](/decisions/adr-003-pivot-policy.md) is revisited.

This is what makes [US-7](/requirements/user-stories.md) — hand a student a `run_id` and both
see identical numbers — actually hold.

## NFR-R5 — Degradation is preferred over wrong answers

The umbrella principle behind R1, R2, and the
[no-partial-results rule](/architecture/request-lifecycle.md#notes-on-specific-steps):

> Under every foreseeable failure, the service **MUST** fail loudly rather than return
> something incomplete or incorrect.

Concretely:

| Situation | Required behaviour |
|---|---|
| One algorithm's output is wrong | Fail the whole run (`500 verification_failed`) |
| One algorithm exceeds the budget | Fail the whole run (`422 run_budget_exceeded`) |
| Registry is inconsistent at startup | Refuse to start |
| Reference sort fails | Fail the run (`500`) |
| Store write fails | Fail the run; do **not** send a response carrying an unretrievable `run_id` |

The alternative at every one of these points — return what completed, flag the rest — produces a
response that looks like a comparison and is not one. [US-4](/requirements/user-stories.md) is
the requirement this serves: a reader must be able to know which numbers are trustworthy.

## What is not specified

* **No availability target.** Meaningless without a deployment topology
  ([NFR overview](/nfr/nfr-overview.md#what-is-missing-and-honestly-so)).
* **No durability target for the result store.** The data is a teaching artefact, and losing a
  run is an inconvenience rather than an incident. A store that loses writes under node failure
  does not violate R1–R5.
* **No multi-region or failover requirement.**