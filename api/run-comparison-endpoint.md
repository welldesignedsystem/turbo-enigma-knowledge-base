---
type: API Endpoint
title: POST /v1/runs
description: Run a submitted integer array through the selected sorting algorithms and return a comparison.
tags: [api, endpoint, runs]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# `POST /v1/runs`

The service's only substantive endpoint. Accepts an array of integers, runs it through the
selected [registered algorithms](/design/algorithm-registry.md), verifies every result, and
returns a comparison.

Implements [FR-1 … FR-11](/requirements/functional-requirements.md). Execution semantics are in
[the run execution model](/design/run-execution-model.md); the request flow is in
[the request lifecycle](/architecture/request-lifecycle.md).

## Synopsis

```
POST /v1/runs
Content-Type: application/json
```

## Request body

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `input` | integer array | **yes** | — | `int64`. May be empty. ≤ 10,000 elements ([FR-2](/requirements/functional-requirements.md)). |
| `algorithms` | string array | no | all registered | Registry IDs. `[]` is rejected; omit for the default ([FR-3](/requirements/functional-requirements.md)). |
| `options.iterations` | integer | no | `1` | 1…1000. Timed repeats; step counts come from one iteration ([FR-10](/requirements/functional-requirements.md)). |
| `options.warmup_iterations` | integer | no | `0` | 0…1000. Executed and discarded. For JIT warm-up only. |
| `options.include_samples` | boolean | no | `false` | Adds `elapsed_samples_ns` to each result. |

Unknown fields are ignored for forward compatibility
([input validation](/design/input-validation.md)). The full rejection table is in
[input validation](/design/input-validation.md#rules-in-evaluation-order).

## Minimal request

```json
{ "input": [5, 3, 8, 1, 9, 2] }
```

Runs all three registered algorithms with defaults.

## Response `201 Created`

```json
{
  "run_id": "run_01JQ8V3K2M7N",
  "created_at": "2026-10-02T09:14:22Z",
  "execution_mode": "synchronous",
  "step_counting_version": "v1",
  "input_length": 6,
  "input_digest": "sha256:3f1c…",
  "options": {
    "iterations": 1,
    "warmup_iterations": 0,
    "include_samples": false
  },
  "results": [
    {
      "algorithm_id": "bubble_sort",
      "output": [1, 2, 3, 5, 8, 9],
      "comparisons": 15,
      "moves": 16,
      "work_units": 31,
      "elapsed_ns": 1183,
      "auxiliary_slots": 0,
      "in_place": true,
      "stable": true,
      "correct": true
    },
    {
      "algorithm_id": "merge_sort",
      "output": [1, 2, 3, 5, 8, 9],
      "comparisons": 10,
      "moves": 32,
      "work_units": 42,
      "elapsed_ns": 2404,
      "auxiliary_slots": 6,
      "in_place": false,
      "stable": true,
      "correct": true
    },
    {
      "algorithm_id": "quicksort",
      "output": [1, 2, 3, 5, 8, 9],
      "comparisons": 8,
      "moves": 30,
      "work_units": 38,
      "elapsed_ns": 971,
      "auxiliary_slots": 0,
      "in_place": true,
      "stable": false,
      "correct": true,
      "pivot_policy": "median_of_three"
    }
  ],
  "disclaimer": "elapsed_ns is environment-dependent and is NOT a valid basis for ranking algorithms. comparisons and moves are deterministic and portable."
}
```

`comparisons` and `moves` above are the
[verified figures for this input](/api/worked-example.md). The `elapsed_ns` values are
**illustrative placeholders**, not measurements — the worked example explains why a real
`elapsed_ns` must never be presented as a portable figure.

## Response fields

| Field | Meaning |
|---|---|
| `run_id` | Unique identifier. Retrievable via [the retrieval endpoint](/api/run-retrieval-endpoint.md). |
| `execution_mode` | `"synchronous"` in phase 1 ([ADR-002](/decisions/adr-002-synchronous-execution.md)). Recorded so a stored run stays interpretable after migration. |
| `step_counting_version` | Which counting ruleset produced the figures ([FR-8](/requirements/functional-requirements.md)). **`v1`** is [the normative rules](/design/step-counting-semantics.md). |
| `input_digest` | Cryptographic hash of the **canonicalised** input, not raw request bytes. Lets two callers confirm they submitted the same array ([US-7](/requirements/user-stories.md)). |
| `results[]` | One entry per algorithm, **in execution order**. |
| `disclaimer` | The [FR-7](/requirements/functional-requirements.md) timing warning. Present in every response, not only in the docs. |

### Per-result fields

| Field | Notes |
|---|---|
| `output` | The sorted array. |
| `comparisons` | **Authoritative.** Portable and reproducible. |
| `moves` | **Authoritative.** A swap is 2. |
| `work_units` | Derived as `comparisons + moves`. **Advisory only** — the 1:1 weighting is arbitrary and it is not authoritative ([ADR-001](/decisions/adr-001-metric-vector-over-scalar.md)). |
| `elapsed_ns` | Median across iterations. Environment-dependent. See [timing measurement](/design/timing-measurement.md). |
| `elapsed_samples_ns` | Present only when `options.include_samples` is true. |
| `auxiliary_slots` | Peak extra array slots. `0` for in-place. |
| `in_place`, `stable` | Copied from the registration so a stored run stays interpretable after a registry change ([data model](/architecture/data-model.md)). |
| `correct` | Always `true` in a `201` response; a mismatch fails the run instead ([FR-6](/requirements/functional-requirements.md)). Present so a stored result is self-describing. |
| `pivot_policy` | Quicksort only ([ADR-003](/decisions/adr-003-pivot-policy.md)). |

## Ordering guarantees

`results[]` is in **registry execution order** (`bubble_sort`, `merge_sort`, `quicksort`),
which is fixed and declared — never sorted by any metric
([FR-4](/requirements/functional-requirements.md)).

The service deliberately does not sort `results[]` by `comparisons`, `moves`, or
`elapsed_ns`. A metric-ordered response would invite the reader to treat the first row as the
best algorithm, which is exactly the conflation this service exists to correct
([US-4](/requirements/user-stories.md)). A client that wants a ranking should compute it from
`comparisons` and state its weighting.

## Examples

### Select a subset

```json
{
  "input": [5, 3, 8, 1, 9, 2],
  "algorithms": ["merge_sort", "quicksort"]
}
```

Bubble sort is skipped. An unknown ID alongside a valid one fails the whole request with
`400` — results are never silently incomplete.

### Find the pivot trap

```json
{
  "input": [7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7],
  "algorithms": ["quicksort", "merge_sort"]
}
```

Quicksort performs **589** comparisons against merge sort's **80** — median-of-three does not
protect against all-duplicate input
([the pivot trap](/algorithms/quicksort.md#the-pivot-trap)).

### Reduce timing noise

```json
{
  "input": [5, 3, 8, 1, 9, 2],
  "options": { "iterations": 200, "warmup_iterations": 50, "include_samples": true }
}
```

Warm-up iterations are discarded. `comparisons` and `moves` still come from **one**
deterministic iteration — repeating the same input cannot change a step count, so averaging
them is meaningless. Only `elapsed_ns` benefits, as the median of 200 samples.

### Empty input

```json
{ "input": [] }
```

Returns `201` with one result per algorithm, all with `comparisons: 0`, `moves: 0`,
`output: []`.

## Responses

| Status | When |
|---|---|
| `201 Created` | Run completed and every result verified. |
| `400 Bad Request` | Malformed body, wrong types, unknown algorithm ID, out-of-range option. |
| `413 Payload Too Large` | Request body over the HTTP cap. |
| `415 Unsupported Media Type` | Not JSON. |
| `422 Unprocessable Content` | Valid but over `max_input_length`, or the per-run time budget was exceeded. |
| `500 Internal Server Error` | Verification failed, or an internal invariant broke. |

Bodies for every non-`201` status follow [the error model](/api/error-model.md).

## Notes and non-obvious behaviour

* **The response is only sent after the run is persisted**
  ([request lifecycle](/architecture/request-lifecycle.md#step-6--persist-before-responding)),
  so a `run_id` in a `201` is always retrievable.
* **No partial results.** One algorithm failing verification fails the whole run.
* **`work_units` is not authoritative.** A client that stores one number should store
  `comparisons` and `moves` and derive `work_units` itself, so its weighting stays visible in
  its own code rather than being inherited invisibly.
* **No caching.** Identical inputs are recomputed, because `elapsed_ns` must reflect a real
  execution and a cache would turn it into a memory-read time.