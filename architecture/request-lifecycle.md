---
type: Architecture Document
title: Request lifecycle
description: The ordered steps from POST /v1/runs to response, including every point where a run can fail.
tags: [architecture, lifecycle, sequence]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Request lifecycle

The sequence for `POST /v1/runs`, and the four places it can fail. Read this when
debugging a surprising result — most surprises come from step 5 or step 7.

## Sequence

```
Client
  │
  │ POST /v1/runs  { input: [...], algorithms: [...], options: {...} }
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. HTTP layer                                                        │
│    - reject oversized request bodies                                │
│    - reject wrong content type                                      │
│    - parse JSON (malformed → 400)                                   │
└─────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Validator — FR-1, FR-2, FR-3                    FAIL → 400 / 422 │
│    - input present, is an array                                     │
│    - may be empty — 0 ≤ length ≤ max_input_length                    │
│    - every element is an integer within int64                        │
│    - no element exceeds 2^53-1 (JS precision ceiling)               │
│    - length ≤ max_input_length (10,000)                             │
│    - algorithms, if present, resolve against the registry           │
└─────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Run orchestrator                                                  │
│    - generate run_id                                                │
│    - canonicalise input → input_digest               FAIL → 500     │
│    - reference_sort(input)  ← computed once, before any algorithm   │
└─────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. For each algorithm, in registry order:                            │
│    4a. working = copy(input)          ← FR-4. never reuse a buffer  │
│    4b. warmup_iterations × { run on a fresh copy; DISCARD metrics }  │
│    4c. iterations × { reset counters; run on a fresh copy;          │
│                      sample elapsed_ns }                            │
│    4d. step counts ← the counters from ONE iteration   FAIL → 500   │
│    4e. verify output == reference_sort                FAIL → 500    │
│    4f. check elapsed against per-run budget          FAIL → 503     │
└─────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. Assemble run document                     FAIL → 500 (write)     │
│    - results[] with metric vectors                                  │
│    - metadata: run_id, input_length, input_digest,                  │
│                created_at, execution_mode, step_counting_version    │
│    - timing disclaimer                              FR-7           │
└─────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. Result store — persist BEFORE responding        FAIL → 500       │
└─────────────────────────────────────────────────────────────────────┘
  │
  ▼
  201 Created + run document
```

## The four failure points

Only four places can fail, and each produces a distinct, named error. This is what keeps the
[error model](/api/error-model.md) small enough to be documented completely.

| Step | Failure | Status | Notable |
|---|---|---|---|
| 2 | Malformed or out-of-contract input | `400` / `422` | Nothing computed. Cheapest failure; always preferred. |
| 3 | Reference sort failed | `500` | Cannot happen for a validated integer array. If it does, the service is broken, and failing here is correct — every later step depends on this value. |
| 4 | An algorithm's output ≠ reference sort | `500` | The run **fails entirely**. No partial results are returned. |
| 6 | Store write failed | `500` | The response is *not* sent, because the client would otherwise hold a `run_id` that cannot be retrieved. |

## Notes on specific steps

### Step 3 — the reference sort is computed first

Computing it once up front, rather than verifying each result as it arrives, means the
comparison target is identical for every algorithm and cannot itself be affected by the
mutation hazard in step 4. It also means a broken reference sort fails the run before any
CPU is spent on the algorithms that depend on it.

### Step 4a — the copy is per iteration, not per algorithm

The buffer is re-copied for every iteration, including warm-up. Warm-up sorts its copy in
place; reusing that buffer for the timed iteration would measure a sorted array and report
best-case numbers ([FR-10](/requirements/functional-requirements.md)). This is the most
common way iteration support produces quietly wrong metrics.

### Step 4c — counters reset per iteration

Only one iteration's counters are reported. Repeating the same input cannot change a step
count, so `comparisons` and `moves` are read from a single deterministic iteration;
`elapsed_ns` is the **median** across iterations. Contract in the
[registry](/design/algorithm-registry.md).

### Step 4d — where the golden numbers are checked

For input `[5,3,8,1,9,2]` this step yields `15`/`16` for bubble sort, `10`/`32` for merge
sort, `8`/`30` for quicksort. Those are regression fixtures for
[phase 1](/roadmap/phase-1-sort-comparison.md), not aspirations. See
[the worked example](/api/worked-example.md).

### Step 4f — the budget exists because bubble sort is quadratic

At the input limit of 10,000 elements, [bubble sort](/algorithms/bubble-sort.md) performs
49,995,000 comparisons. A budget is required so that a pathological but *valid* input
cannot hold a request open indefinitely ([FR-11](/requirements/functional-requirements.md)).

### Step 6 — persist before responding

Writing first and responding second costs a few milliseconds and buys a guarantee that a
`run_id` in a client is always retrievable. The reverse order produces the worst class of
bug in an API: a successful response containing an identifier that 404s.

## What is deliberately absent

* **No retries.** Retrying a comparison run produces the same deterministic result; there is
  nothing transient to recover from. Retries belong to the store write in a future phase.
* **No partial results.** If any algorithm fails verification, the whole run fails. A
  comparison missing one of its three algorithms is not a comparison.
* **No cancellation.** Phase 1 has no asynchronous handle to cancel
  ([ADR-002](/decisions/adr-002-synchronous-execution.md)). The time budget in step 4f is
  the only bound.

## Retrieval path

`GET /v1/runs/{run_id}` is deliberately much simpler: resolve the ID in the
[result store](/architecture/data-model.md), return the stored document verbatim, or `404`.

It **MUST NOT** recompute anything. Recomputation would silently change historical results
whenever an algorithm's implementation changed, which would make stored runs worthless as
the reproducible reference that [US-7](/requirements/user-stories.md) depends on. The
`step_counting_version` in the stored document is what lets a consumer interpret an old
result correctly.