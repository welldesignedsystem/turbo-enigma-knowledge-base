---
type: Requirements Document
title: Functional requirements
description: The normative FR-1 to FR-12 contract for the Algorithm Run Service, phase 1.
tags: [requirements, functional, contract]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Functional requirements

Normative for phase 1. `MUST` / `MUST NOT` are binding on the implementation; `SHOULD`
permits documented deviation. Requirement IDs are stable — cite them, do not renumber.

Related: [product brief](/requirements/product-brief.md),
[scope and phasing](/requirements/scope-and-phasing.md),
[the API contract](/api/index.md).

---

## FR-1 — Accept an integer array

`POST /v1/runs` **MUST** accept a JSON body containing `input`, a non-empty array of
integers. Integers **MUST** be 64-bit signed (`int64`), inclusive range
`-9,223,372,036,854,775,808` to `9,223,372,036,854,775,807`.

The service **MUST** reject non-integer values, fractional numbers, booleans, strings,
`null`, and non-finite floats (`NaN`, `±Infinity`) with `400`. Booleans **MUST** be
rejected even where a host language coerces them silently, because `true` is not 1 as a
type and accepting it teaches the wrong thing.

> **Portability note.** `int64` is the service's contract, not the host language's. A
> JavaScript client **MUST NOT** send integers beyond
> `Number.MAX_SAFE_INTEGER` (2⁵³−1 = 9,007,199,254,740,991); `JSON.parse` silently loses
> precision above that, so a round-tripped array will not match what the caller sent. The
> service **SHOULD** reject any integer exceeding 2⁵³−1 with `400` and a message naming
> JavaScript's precision limit, rather than returning results for an input the caller cannot
> reproduce.

## FR-2 — Enforce the input length limit

The service **MUST** reject arrays longer than the configured `max_input_length` (default
10,000) with `422`, per [input validation](/design/input-validation.md).

The limit **MUST** be enforced because of [bubble sort](/algorithms/bubble-sort.md), which
at n = 10,000 performs 49,995,000 comparisons. The service **MUST NOT** raise the default
limit to accommodate quicksort without also raising the phase-1 latency budget, because
bubble sort remains the dominant cost.

Empty arrays (`input: []`) **MUST** be accepted and **MUST** return a successful empty
result for every selected algorithm with zero metrics. A single-element array **MUST** also
succeed with zero comparisons and zero moves.

## FR-3 — Select algorithms, defaulting to all

The body **MAY** carry `algorithms`, an array of registry IDs. When absent, the service
**MUST** run every registered algorithm. An empty `algorithms` array **MUST** be rejected
with `400` (use omission to mean "all").

An unknown algorithm ID **MUST** fail the whole request with `400` and an error naming the
unknown IDs, rather than silently skipping them. Silently dropping an algorithm would make
a comparison silently incomplete, which is worse than a visible error.

## FR-4 — Run each algorithm on an independent copy

Each selected algorithm **MUST** receive its own defensive copy of the input array. The
service **MUST NOT** pass a previously-run algorithm's output to the next algorithm.

This is a correctness requirement, not a performance nicety. Bubble sort and quicksort sort
in place; without the copy, the second algorithm receives an already-sorted array and
reports best-case numbers for input that was not best-case. The comparison would be
silently, systematically wrong.

Algorithms **MUST** be invoked in a deterministic order (registry declaration order) so that
results are reproducible.

## FR-5 — Return the metric vector per algorithm

Each result **MUST** contain, per algorithm:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `comparisons` | integer ≥ 0 | yes | Pairwise key comparisons performed. |
| `moves` | integer ≥ 0 | yes | Single-element relocations; a swap is 2. |
| `elapsed_ns` | integer ≥ 0 | yes | Wall-clock nanoseconds. Environment-dependent. |
| `work_units` | integer ≥ 0 | yes | Derived: `comparisons + moves`. Advisory only. |
| `auxiliary_slots` | integer ≥ 0 | when applicable | Peak extra array slots used. |
| `output` | integer array | yes | The sorted array. |
| `correct` | boolean | yes | Output matched the reference sort. |

Counting rules are normative and specified in
[step-counting semantics](/design/step-counting-semantics.md). `work_units` **MUST** be
marked advisory in the API documentation and **MUST NOT** be the field a caller is steered
toward. See [ADR-001](/decisions/adr-001-metric-vector-over-scalar.md).

## FR-6 — Verify output before returning

Before returning a run, the service **MUST** compute a reference sort of the original input
independently and **MUST** compare each algorithm's output against it.

If any algorithm's output differs, the service **MUST NOT** return a successful response.
It **MUST** fail the run with `500` and an error identifying the algorithm, per the
[error model](/api/error-model.md). A wrong sort is the single most damaging output this
service could produce, because its entire purpose is trustworthy comparison.

The reference sort **MUST** be a different implementation family from the registered ones
where practical, so that a bug shared between two algorithms is not self-confirming.

## FR-7 — Label timing honestly

The response **MUST** carry a top-level note that `elapsed_ns` is environment-dependent and
not a valid basis for ranking algorithms. The service **MUST NOT** present a sorted
"fastest to slowest" ranking computed from `elapsed_ns`.

`elapsed_ns` is included because callers ask for it, not because it is sound. Rationale and
the specific confounds are in [timing measurement](/design/timing-measurement.md).

## FR-8 — Report run metadata

Every response **MUST** include:

* `run_id` — stable unique identifier.
* `input_length` and a `input_digest` — a cryptographic hash of the input, so two callers
  can confirm they submitted the same array and so results can be correlated. The digest
  **MUST** be computed over a canonical serialisation, not the raw request bytes.
* `created_at` — ISO 8601 with offset.
* `execution_mode` — `"synchronous"` in phase 1, per
  [ADR-002](/decisions/adr-002-synchronous-execution.md).
* `step_counting_version` — the identifier of the counting ruleset applied. When the
  semantics in [step-counting semantics](/design/step-counting-semantics.md) change, this
  changes, and historical results remain interpretable because the version is recorded.

The version field is the mechanism that keeps old numbers meaningful. Its absence is the
reason most published benchmark corpora become uninterpretable within a year.

## FR-9 — Expose the algorithm registry

`GET /v1/algorithms` **MUST** return every registered algorithm with its ID, display name,
complexity bounds, whether it is in-place, auxiliary space class, stability, and the
`step_counting_version` it was built against.

Rationale: a caller that does not know which pivot policy a `quicksort` implementation
uses cannot interpret its worst case. The registry **MUST** therefore publish
`pivot_policy` for quicksort, per [ADR-003](/decisions/adr-003-pivot-policy.md).

## FR-10 — Support iterations and warm-up

The body **MAY** carry `options.iterations` (default 1, max 1000) and
`options.warmup_iterations` (default 0, max 1000).

With `iterations > 1`, the service **MUST** report `comparisons` and `moves` from a
single deterministic iteration — repeating the *same* input a thousand times cannot change
a step count, so averaging them is meaningless — and **MUST** report `elapsed_ns` as the
**median** of the timed iterations, not the mean and not the minimum.

Median rather than mean because a single GC pause or context switch would otherwise skew
the reported figure; median rather than minimum because the minimum hides real cost rather
than removing noise. The full sample **SHOULD** be available via
`options.include_samples` for callers who want to do their own statistics.

Warm-up iterations **MUST** be discarded entirely — never included in reported metrics —
and exist only to give JIT-compiled runtimes time to compile.

## FR-11 — Reject work exceeding the time budget

The service **MUST** apply a per-run wall-clock budget. If the run exceeds it, it **MUST**
abort and return `503` or `422` with a message naming the budget, per the
[error model](/api/error-model.md).

The service **MUST NOT** silently return partial results. A truncated comparison is
indistinguishable from a real one.

## FR-12 — Validate output against input, not against a previous run

Output verification under FR-6 **MUST** compare against a fresh reference sort of the
*submitted* input on every run. The service **MUST NOT** cache a reference sort keyed only
by `input_digest` without also validating the cache against the current input.

A cache keyed on a digest collision, or a cache populated under the buggy mutation
described in FR-4, would convert a caught bug into a persistent wrong answer.

---

## Traceability

| Requirement | Satisfied by | Verified by |
|---|---|---|
| FR-1 input contract | [input validation](/design/input-validation.md) | Validation unit tests, phase 1 |
| FR-2 length limit | [input validation](/design/input-validation.md) | Boundary tests at n−1, n, n+1 |
| FR-3 algorithm selection | [algorithm registry](/design/algorithm-registry.md) | Registry contract tests |
| FR-4 defensive copy | [run execution model](/design/run-execution-model.md) | Mutation regression test |
| FR-5 metric vector | [step-counting semantics](/design/step-counting-semantics.md) | Golden-number tests vs [worked example](/api/worked-example.md) |
| FR-6 output verification | [run execution model](/design/run-execution-model.md) | Injected-fault test |
| FR-7 timing labelling | [timing measurement](/design/timing-measurement.md) | Response-shape test |
| FR-8 run metadata | [data model](/architecture/data-model.md) | Schema test |
| FR-9 registry endpoint | [run retrieval endpoint](/api/run-retrieval-endpoint.md) | Contract test |
| FR-10 iterations | [timing measurement](/design/timing-measurement.md) | Determinism test |
| FR-11 time budget | [reliability NFR](/nfr/reliability.md) | Load test |
| FR-12 verification integrity | [run execution model](/design/run-execution-model.md) | Cache-poisoning test |

No automated verification exists yet — this service is not implemented. The table records
what *must* be tested when it is, not what has been tested.