---
type: API Endpoint
title: GET /v1/runs/{run_id}
description: Retrieve a previously computed run by identifier, plus the registry and health endpoints.
tags: [api, endpoint, retrieval, registry]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# `GET /v1/runs/{run_id}`

Returns a previously computed run, verbatim. Supports
[US-7](/requirements/user-stories.md): an instructor hands a student a `run_id` and both see
identical numbers.

## Synopsis

```
GET /v1/runs/{run_id}
```

## Response `200 OK`

The stored `Run` document, byte-for-byte as it was returned by
[`POST /v1/runs`](/api/run-comparison-endpoint.md#response-201-created).

```json
{
  "run_id": "run_01JQ8V3K2M7N",
  "created_at": "2026-10-02T09:14:22Z",
  "execution_mode": "synchronous",
  "step_counting_version": "v1",
  "input_length": 6,
  "input_digest": "sha256:3f1c…",
  "input": [5, 3, 8, 1, 9, 2],
  "options": { "iterations": 1, "warmup_iterations": 0, "include_samples": false },
  "results": [
    { "algorithm_id": "bubble_sort", "output": [1,2,3,5,8,9], "comparisons": 15, "moves": 16,
      "work_units": 31, "elapsed_ns": 1183, "auxiliary_slots": 0,
      "in_place": true, "stable": true, "correct": true }
  ],
  "disclaimer": "elapsed_ns is environment-dependent and is NOT a valid basis for ranking algorithms. comparisons and moves are deterministic and portable."
}
```

`input` is included here though it was absent from the `POST` response, so a stored run is
self-contained and auditable ([data model](/architecture/data-model.md#on-storing-input)).

## The one rule this endpoint exists to enforce

> **Retrieval MUST NOT recompute anything.**

Recomputing would silently rewrite history every time an algorithm's implementation changed.
A `quicksort` whose [pivot policy](/decisions/adr-003-pivot-policy.md) is revised in phase 2
would, under recomputation, change the stored answer to an old `run_id` from a year ago. The
run would still claim to be the same run, and would now report figures from an algorithm that
never produced them.

This is why `Result` copies `in_place`, `stable`, and `pivot_policy` from the registration at
run time instead of referencing it ([data model](/architecture/data-model.md#result)) — a
stored run must describe the algorithm that actually ran, not the one currently registered.

`step_counting_version` in the stored document is the other half of the guarantee: it tells a
consumer which ruleset produced the numbers, so an old result remains interpretable after
[the counting rules change](/design/step-counting-semantics.md#versioning-this-document).

## Responses

| Status | When |
|---|---|
| `200 OK` | Run found. |
| `400 Bad Request` | `run_id` is not a well-formed identifier. |
| `404 Not Found` | No run with that identifier. |

`404` rather than `403`: phase 1 has no authentication, so there is no distinction to draw
between "does not exist" and "not yours" ([security](/nfr/security.md)).

## Notes

* Retrieval is **cheap and unbounded** — a run lookup is a single store read, with none of
  the CPU cost of executing algorithms. This asymmetry is why browsing historical runs is
  safe while submitting new ones is not
  ([scalability](/nfr/scalability.md)).
* There is **no listing endpoint** in phase 1
  ([scope and phasing](/requirements/scope-and-phasing.md)). Retrieval is by identifier only.
* A `run_id` is not guessable, but phase 1 makes no security claim about that
  ([security](/nfr/security.md#what-phase-1-does-not-provide)).

---

# `GET /v1/algorithms`

Publishes the [registry](/design/algorithm-registry.md) ([FR-9](/requirements/functional-requirements.md)).

```
GET /v1/algorithms
```

## Response `200 OK`

```json
{
  "step_counting_version": "v1",
  "algorithms": [
    {
      "id": "bubble_sort",
      "name": "Bubble sort",
      "in_place": true,
      "stable": true,
      "auxiliary_space": "O(1)",
      "best": "O(n)",
      "average": "O(n^2)",
      "worst": "O(n^2)",
      "known_limitations": [],
      "step_counting_version": "v1"
    },
    {
      "id": "merge_sort",
      "name": "Merge sort",
      "in_place": false,
      "stable": true,
      "auxiliary_space": "O(n)",
      "best": "O(n log n)",
      "average": "O(n log n)",
      "worst": "O(n log n)",
      "known_limitations": [],
      "step_counting_version": "v1"
    },
    {
      "id": "quicksort",
      "name": "Quicksort (Lomuto partition, median-of-three pivot)",
      "in_place": true,
      "stable": false,
      "auxiliary_space": "O(log n)",
      "best": "O(n log n)",
      "average": "O(n log n)",
      "worst": "O(n^2)",
      "pivot_policy": "median_of_three",
      "known_limitations": [
        "All-duplicate input degrades to O(n^2): 496 comparisons at n=32, versus 32 for merge sort. Median-of-three does not protect against this. A 3-way (Dutch national flag) partition is the standard fix.",
        "Worst case also triggers on already-sorted and reverse-sorted input if the pivot policy is ever changed to first/last element; the current policy reduces those to 103 and 126 comparisons respectively."
      ],
      "step_counting_version": "v1"
    }
  ]
}
```

## Why `pivot_policy` and `known_limitations` are mandatory

`worst: "O(n^2)"` on its own is close to useless — it invites a reader to average-case-plan
around `O(n log n)`. Publishing the pivot policy and the concrete failure input is what makes
the bound actionable: a caller learns both that the cliff exists and which input triggers it.

The measured figures in `known_limitations` (496, 103, 126, 32) come from
[the pivot trap](/algorithms/quicksort.md#the-pivot-trap); their provenance is in
[measurement provenance](/references/measurement-provenance.md).

---

# Health endpoints

| Endpoint | Meaning |
|---|---|
| `GET /healthz` | Process is alive. No dependency checks. |
| `GET /readyz` | Process can serve traffic — the [result store](/architecture/data-model.md) is reachable. |

`healthz` deliberately does **not** check the store. A liveness probe that fails when a
dependency is down causes a restart loop during a store outage, turning a degradation into an
outage. `readyz` is where the dependency belongs.

Both return `200` with a minimal JSON body, or `503` with the
[error envelope](/api/error-model.md).