---
type: Design Document
title: Error model
description: Status codes, the error envelope, and the error codes the service returns.
tags: [api, errors, contract]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Error model

Every non-`2xx` response uses one envelope. Errors are designed to be actionable without
reading the source ([US-6](/requirements/user-stories.md)): each names the offending field,
its value, and the applicable limit.

## Envelope

```json
{
  "error": {
    "code": "input_too_large",
    "message": "input contains 10001 elements; the maximum is 10000.",
    "details": {
      "field": "input",
      "received": 10001,
      "limit": 10000,
      "reason": "The limit bounds the cost of bubble_sort, which performs n(n-1)/2 comparisons."
    },
    "run_id": null,
    "request_id": "req_01JQ8V3K9T4B"
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `code` | yes | Stable, machine-readable, `snake_case`. Safe to branch on. |
| `message` | yes | One sentence, human-readable. Not localised in phase 1. |
| `details` | yes | Structured. Key varies by `code`; see below. |
| `run_id` | no | Present when the failure occurred after a run was created. Lets an operator correlate a failed run. |
| `request_id` | yes | Present on every response, success and failure. The correlation handle for logs. |

`code` is the contractual field. `message` is for humans and **may** change without notice;
do not parse it.

## Codes

| `code` | Status | Cause | `details` keys |
|---|---|---|---|
| `malformed_json` | `400` | Body is not parseable JSON. | — |
| `body_not_object` | `400` | Body is valid JSON but not an object. | — |
| `missing_field` | `400` | A required field is absent. | `field`, `expected` |
| `invalid_type` | `400` | Wrong type, e.g. `input` is not an array. | `field`, `expected`, `received` |
| `not_an_integer` | `400` | An element is not an integer. | `field`, `index`, `value`, `received_type` |
| `value_out_of_range` | `400` | Element outside `int64`. | `field`, `index`, `value`, `min`, `max` |
| `exceeds_safe_integer` | `400` | Element above 2⁵³−1 (JavaScript precision ceiling). | `field`, `index`, `value`, `limit` |
| `unsupported_media_type` | `415` | `Content-Type` is not JSON. | `received` |
| `payload_too_large` | `413` | Body over the HTTP cap. | `limit`, `received` |
| `unknown_algorithm` | `400` | An ID is not in the registry. | `field`, `unknown`, `available` |
| `empty_algorithm_list` | `400` | `algorithms: []`. | `field`, `hint` |
| `invalid_option` | `400` | `options` value out of range or wrong type. | `field`, `value`, `min`, `max` |
| `input_too_large` | `422` | Over `max_input_length`. | `field`, `received`, `limit`, `reason` |
| `run_budget_exceeded` | `422` | Per-run time budget exceeded. | `budget_ms`, `algorithms` |
| `run_not_found` | `404` | No such `run_id`. | `field` |
| `invalid_run_id` | `400` | Malformed `run_id`. | `field`, `value` |
| `verification_failed` | `500` | An algorithm's output did not match the reference sort. | `algorithm_id` |
| `internal_error` | `500` | Unexpected internal failure. | `run_id` |

## Status code selection

| Status | Meaning here |
|---|---|
| `400` | The request is malformed or violates the contract. **Do not retry unchanged.** |
| `404` | The addressed resource does not exist. |
| `413` | The body exceeded a transport-level cap. |
| `415` | The body is well-formed but not a supported media type. |
| `422` | The request is **well-formed and valid** but the service cannot process it as asked — too large, or over the time budget. Distinguishing this from `400` lets a client separate "fix your code" from "ask for less". |
| `500` | The service is broken, not the request. Retryable only with backoff, and pointless if the cause is deterministic. |

The `400`/`422` split is the one that matters most to integrators. A `400` means the request
will never succeed; a `422` means it *could* succeed with a smaller input.

## The two 500s that matter

### `verification_failed`

[FR-6](/requirements/functional-requirements.md) requires the run to fail rather than return
a wrong sort. This is the one error a client should **never** retry and should **always**
report: it means the service produced or would have produced incorrect data.

```json
{
  "error": {
    "code": "verification_failed",
    "message": "quicksort output did not match the reference sort; the run has been discarded.",
    "details": { "algorithm_id": "quicksort", "run_id": "run_01JQ8V3K2M7N" },
    "run_id": "run_01JQ8V3K2M7N",
    "request_id": "req_01JQ8V3K9T4B"
  }
}
```

The run is discarded entirely. No partial results, no `results[]` with one entry
flagged — see [the verification gate](/design/run-execution-model.md#decision-2--the-verification-gate).

### `run_budget_exceeded`

[FR-11](/requirements/functional-requirements.md) requires aborting rather than returning
partial results. `details.algorithms` names the algorithms that had not completed, so a
caller can resubmit a smaller input or a narrower algorithm selection and know what to expect.

`422` rather than `503`: the service is healthy and the request was merely too expensive for
the budget. `503` would wrongly suggest the caller should retry later unchanged.

## Error bodies must not leak internals

An `internal_error` **MUST NOT** include stack traces, file paths, database errors, or
registry internals in `message` or `details`. The correlation handle is `request_id`, and the
detail belongs in server logs
([observability](/nfr/observability.md)).

This is a specific risk in this service because a failure inside an algorithm can surface a
host-language message containing buffer contents — that would put a caller's input array into
a response body, and potentially into an intermediary's logs.

## Retry guidance to publish

| Status | Retry? |
|---|---|
| `400`, `404`, `413`, `415`, `422` | **No.** The request is wrong; retrying unchanged cannot help. |
| `500` `verification_failed` | **No.** Deterministic; retrying repeats the defect. |
| `500` `internal_error` | Maybe, with exponential backoff. Still worth reporting. |
| `503` from `readyz` | Yes, after the dependency recovers. |