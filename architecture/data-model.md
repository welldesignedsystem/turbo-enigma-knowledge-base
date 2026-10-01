---
type: Architecture Document
title: Data model
description: The entities the Algorithm Run Service persists, their fields, invariants, and lifecycles.
tags: [architecture, data-model, entities]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Data model

Four entities. Three are read-only for phase 1 callers; the fourth is a registry snapshot.

## Entity diagram

```
    AlgorithmRegistration          1
              │
              │ selected by
              ▼
  ┌──────────────────┐
  │       Run        │  1
  └────────┬─────────┘
           │ contains
           ▼  n
  ┌──────────────────┐
  │      Result      │  n
  └──────────────────┘

  ReferenceSort  ── transient, not persisted (see below)

  InputDigest    ── derived, stored on Run
```

## `Run`

The unit of work and retrieval. Written once, never updated, never deleted in phase 1.

| Field | Type | Notes |
|---|---|---|
| `run_id` | string | Unique. Primary key. |
| `created_at` | timestamp | ISO 8601 with offset ([FR-8](/requirements/functional-requirements.md)). |
| `input_length` | integer | `input.length`. |
| `input_digest` | string | Cryptographic hash of the **canonical** input serialisation. |
| `input` | integer array | Stored so a run is self-contained and verifiable. |
| `algorithms_requested` | string array | As supplied, or the full resolved default. |
| `execution_mode` | string | `"synchronous"` in phase 1 ([ADR-002](/decisions/adr-002-synchronous-execution.md)). |
| `step_counting_version` | string | The counting ruleset applied. |
| `options` | object | `iterations`, `warmup_iterations`, `include_samples`. |
| `results` | array of [`Result`](#result) | One per algorithm, in execution order. |
| `disclaimer` | string | The FR-7 timing warning. |

### Invariants

* `run_id` is immutable and unique.
* `results` is never modified after write. `GET /v1/runs/{run_id}` returns the stored
  document verbatim and never recomputes — see
  [the retrieval path](/architecture/request-lifecycle.md#retrieval-path).
* Every `Result` in `results` was verified against the reference sort before the `Run` was
  written. A stored `Run` therefore contains no unverified results.

### On storing `input`

Storing the input is a size/latency trade rather than a free win. Without it a run cannot
be reproduced or audited, and `input_digest` would be a hash of something no consumer can
see. Against that, the array is bounded by
[the input limit](/design/input-validation.md) to 10,000 integers, which is small enough to
store inline.

> **Operationally:** storing arrays inline inflates the store quickly and makes per-row
> scans useless. Phase 2 should move `input` and `results[]` to object storage and keep
> only the `Run` shell plus pointers in the primary store. Recorded in
> [the risk register](/risks/risk-register.md).

### On `input_digest`

Computed over a **canonical serialisation** of the array, not raw request bytes. Raw bytes
would make the digest differ for two semantically identical requests that differ only in
whitespace or key order, defeating the purpose of the field
([FR-8](/requirements/functional-requirements.md)).

Canonical form is versioned alongside `step_counting_version`. If the canonicalisation ever
changes, previously computed digests must not be silently reinterpreted — two arrays would
collide or a single array would produce two different digests for the same logical input.
A digest is a grouping key, and a grouping key that changes meaning retroactively has
silently broken every historical comparison.

## `Result`

One algorithm's outcome within a run.

| Field | Type | Notes |
|---|---|---|
| `algorithm_id` | string | Registry ID, e.g. `bubble_sort`. |
| `output` | integer array | The sorted array. |
| `comparisons` | integer ≥ 0 | From a single deterministic iteration ([FR-10](/requirements/functional-requirements.md)). |
| `moves` | integer ≥ 0 | Swap = 2. See [step-counting semantics](/design/step-counting-semantics.md). |
| `work_units` | integer ≥ 0 | Derived, advisory. **Not** authoritative ([ADR-001](/decisions/adr-001-metric-vector-over-scalar.md)). |
| `elapsed_ns` | integer ≥ 0 | Median across iterations. Environment-dependent. |
| `elapsed_samples_ns` | integer array | Present when `options.include_samples` is true. |
| `auxiliary_slots` | integer ≥ 0 | `0` for in-place algorithms. |
| `correct` | boolean | Always `true` in a stored run; a false value fails the run instead. |
| `in_place` / `stable` | boolean | Copied from the registration so a stored run stays interpretable after the registry changes. |

`algorithm_id` is **not** a foreign key that can be resolved later. The behavioural flags
are copied onto the `Result` deliberately: if a future release changes
[quicksort's pivot policy](/decisions/adr-003-pivot-policy.md), an old stored run must still
describe the algorithm that actually ran. Resolving against the current registry would
reinterpret history.

## `ReferenceSort`

Computed in-process per [FR-6](/requirements/functional-requirements.md) and **not
persisted**. It is an internal verification input, not a result.

It is intentionally *not* included in `results[]` either. A reference sort among the
comparison results would suggest it is one of the compared algorithms, which it is not —
it is the thing the others are checked against.

## `AlgorithmRegistration`

The registry snapshot, published by `GET /v1/algorithms` ([FR-9](/requirements/functional-requirements.md)).

| Field | Type | Notes |
|---|---|---|
| `id` | string | Registry key. Stable within a `step_counting_version`. |
| `name` | string | Display name. |
| `stability` | enum | `stable`, `unstable`. All three phase-1 algorithms are stable. |
| `in_place` | boolean | |
| `auxiliary_space` | string | `O(1)`, `O(n)`, `O(log n)`. |
| `best` / `average` / `worst` | string | Complexity classes. |
| `pivot_policy` | string | Quicksort only. **Required** — see [ADR-003](/decisions/adr-003-pivot-policy.md). |
| `known_limitations` | string array | Not optional in practice. Quicksort's duplicate-key cliff is listed here. |
| `step_counting_version` | string | Must match the counting ruleset the implementation uses. |

`known_limitations` exists because a registry that publishes only complexity classes is
actively misleading: `O(n log n)` average with an `O(n²)` worst case invites a caller to
treat the average as a guarantee. Publishing the cliff is what
[US-3](/requirements/user-stories.md) is built on.

## Lifecycles

| Entity | Created | Read | Updated | Deleted |
|---|---|---|---|---|
| `Run` | On run completion, before response | By `run_id` | Never | Retention policy, unspecified in phase 1 |
| `Result` | With its `Run` | Via its `Run` | Never | With its `Run` |
| `AlgorithmRegistration` | At startup from compiled-in code | Via `GET /v1/algorithms` | Never | n/a |
| `ReferenceSort` | Per run | Within the run only | Never | Immediately |

Retention is deliberately unspecified: phase 1 has no volume, so choosing a policy now would
be a guess. It is tracked in [the risk register](/risks/risk-register.md).