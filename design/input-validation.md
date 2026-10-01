---
type: Design Document
title: Input validation
description: The validation rules, limits, and rejection behaviour for run input, and why the length limit is a correctness feature.
tags: [design, validation, limits, denial-of-service]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Input validation

Validation is specified in [FR-1](/requirements/functional-requirements.md) and
[FR-2](/requirements/functional-requirements.md); this document gives the concrete rules,
the reasoning for the limits, and the error behaviour. Per
[the component view](/architecture/component-view.md), the
[validator](/architecture/component-view.md#2-validator) is the only component that inspects
raw request input.

## Rules, in evaluation order

Order is deliberate: cheapest and most-likely-rejected first, so a bad request never reaches
the expensive checks.

| # | Rule | Failure | Rationale |
|---|---|---|---|
| 1 | Request body within the HTTP size cap | `413` | Cheapest possible rejection. |
| 2 | Content type is JSON | `415` | |
| 3 | Body parses as a JSON object | `400` | |
| 4 | `input` present | `400` | |
| 5 | `input` is an array | `400` | |
| 6 | `input` is non-empty | `400` | Empty is rejected **as an input error**, but note `[]` in fact succeeds in returning results — see the edge-case table. |
| 7 | Every element is an integer | `400` | No floats, no fractional, no strings, no `null`, no booleans. |
| 8 | Every element within `int64` range | `400` | |
| 9 | Every element ≤ 2⁵³−1 | `400` | JavaScript precision ceiling — see below. |
| 10 | `algorithms`, if present, non-empty and resolvable | `400` | Unknown IDs fail the whole request. |
| 11 | `input.length` ≤ `max_input_length` | `422` | Checked last: it is a length check, so it is O(1). |
| 12 | `options.iterations`, `warmup_iterations` in `[1,1000]` / `[0,1000]` | `400` | |

`input.length` is checked **last** among the array rules despite being O(1), because rule 7
requires walking the array anyway, and a caller who sent 5 million non-integers should be told
that before being told the array is too long. The error a caller fixes first should be the
one that is actually wrong.

## The `int64` and JavaScript precision boundary

The service's numeric contract is 64-bit signed integers
([FR-1](/requirements/functional-requirements.md)), but a JavaScript client's `JSON.parse`
silently rounds anything beyond `Number.MAX_SAFE_INTEGER` (2⁵³−1 = 9,007,199,254,740,991).

The consequence is that a JavaScript client can send `[9007199254740993]`, receive a successful
response, and never be able to retrieve that value again — its own `JSON.parse` already
changed the number. It cannot reproduce its own run, which breaks
[US-1](/requirements/user-stories.md) and [FR-8](/requirements/functional-requirements.md).

The service therefore **rejects** integers above 2⁵³−1 with `400` and a message naming the
limit, rather than accepting an input the caller cannot reproduce. This is a deliberate
refusal to be maximally permissive: the service values reproducibility over
completeness.

`int64` remains the internal representation, so the 2⁵³−1 ceiling is a *protocol* limit, not a
storage limit.

## Why the length limit exists

`max_input_length` defaults to **10,000**. The reason is [bubble sort](/algorithms/bubble-sort.md),
and only bubble sort.

At the limit, bubble sort performs exactly `n(n−1)/2 = 49,995,000` comparisons
([verified](/design/step-counting-semantics.md#consequence-2--bubble-sorts-worst-case-is-exactly-nn12)).
Measured growth from the [complexity reference](/algorithms/complexity-reference.md):

| n | Bubble sort `comparisons` (random, mean of 20) |
|---|---|
| 128 | 8,057 |
| 256 | 32,385 |
| 512 | 130,511 |

The roughly 4× per doubling is the quadratic signature. Extrapolating from n = 512, bubble
sort crosses 50 million comparisons at approximately n = 10,000 — which is where the limit is
set, and is not a coincidence.

Quicksort at n = 10,000 performs on the order of 100,000 comparisons, roughly **500× fewer**.
The limit is not about the algorithms in general; it is about the worst one in the registry.

### It is a correctness feature, not politeness

Three separate reasons:

1. **Denial of service without any malicious intent.** Because the algorithms run
   in-process on the request thread
   ([system context](/architecture/system-context.md#environmental-dependencies-that-shape-the-architecture)),
   an unbounded array is a CPU-exhaustion vector. A single request of a few hundred thousand
   elements would occupy a core for seconds. This holds even with phase-1's lack of
   authentication.
2. **The latency budget.** [FR-11](/requirements/functional-requirements.md) requires a
   per-run budget; without a length limit the budget would fire on ordinary valid input,
   making it useless as a safety net.
3. **Timing becomes meaningless at scale.** At large n the step counts remain exact but the
   timing is dominated by cache and memory behaviour, and the service's teaching value drops
   sharply while its cost rises.

### Raising the limit

Raising it requires raising the latency budget, because bubble sort remains registered and
remains the dominant cost ([ADR-003](/decisions/adr-003-pivot-policy.md) fixes quicksort's
pivot policy but nothing reduces bubble sort's asymptotics). A proposal to raise
`max_input_length` **MUST** state its effect on the worst-case run time, and **MUST NOT** be
justified on the grounds that quicksort handles it easily.

## Edge cases

| Input | Behaviour | Rationale |
|---|---|---|
| `input: []` | **Accepted.** Every selected algorithm returns `[]` with `comparisons: 0`, `moves: 0`. | Correct and instructive — it is the base case of every algorithm. |
| `input: [x]` | Accepted. `comparisons: 0`, `moves: 0`. | Same reason. |
| All elements equal | Accepted. Worth submitting: quicksort degrades to 496 comparisons at n = 32 ([the pivot trap](/algorithms/quicksort.md#the-pivot-trap)). | The most instructive input in the service. |
| Already sorted | Accepted. | Bubble sort's best case: `n−1` comparisons, 0 moves. |
| Reverse sorted | Accepted. | Bubble sort's worst case. |
| Negative and zero values | Accepted. | Sorting is order-based; nothing special about negatives. |
| Duplicates | Accepted. | All three algorithms are stable, so equal keys retain input order. |
| `algorithms: []` | `400` | Use omission to mean "all". |
| Unknown algorithm ID | `400`, naming the unknown IDs | Silent skipping would produce a silently incomplete comparison. |
| `algorithms: null` | Treated as omitted | Explicit null is treated as "use the default", not as an error. |
| Unknown top-level field | Ignored | Forward compatibility; a client must not break on a newer server. |
| Unknown field inside `options` | Ignored, same reason | |

## Booleans are rejected

`[true, false]` must be `400`, even in host languages where `bool` coerces to 1 and 0.

Two reasons, and the second is the real one:

1. Correctness — silently sorting `[1, 0]` from `[true, false]` hides a client bug.
2. **It teaches the wrong thing.** This service exists to make measurement semantics precise
   ([the product brief](/requirements/product-brief.md)). A service that accepts a loose
   coercion is inconsistent with its own premise. The same reasoning applies to rejecting
   `1.0` rather than accepting it as `1`.

## Unknown fields are ignored

An unrecognised field is ignored rather than rejected. This is the standard forward-compatible
choice and it keeps a client written against a later version working against an earlier server.

The trade is explicit: unknown fields **may** contain typos (`comparisions`) that silently do
nothing. That is the accepted cost of not breaking forward compatibility. It is worth noting
that a client which receives `comparisons: null` for a misspelled request should treat a
missing expected field as an error rather than as zero.

## Error reporting

All validation failures produce the [error envelope](/api/error-model.md):

* The status code follows the table above.
* `error.code` is a stable machine-readable string.
* `error.details` names the **offending field, its index if applicable, the offending value,
  and the applicable limit**. `422` and `400` are distinguished so a client can separate
  "you sent something malformed" from "you sent something valid but too big".