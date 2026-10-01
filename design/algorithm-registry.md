---
type: Design Document
title: Algorithm registry
description: The contract an algorithm must satisfy to be registered, and why algorithms are compiled in rather than uploaded.
tags: [design, registry, extension-point]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Algorithm registry

The registry is the single source of truth for *which algorithms exist, in what order they
run, and what their properties are*. It feeds three consumers, all of which must agree:

1. Default algorithm selection when the request omits `algorithms`
   ([FR-3](/requirements/functional-requirements.md)).
2. `GET /v1/algorithms` ([FR-9](/requirements/functional-requirements.md)).
3. Execution order ([FR-4](/requirements/functional-requirements.md)).

Deriving all three from one table is the entire point. Three separate lists will drift, and
the drift will show up as an API that advertises an algorithm it will not run, or a comparison
whose reported order is not the order the numbers were produced in.

## Phase 1 registry

| Order | `id` | In-place | Stable | Best | Average | Worst | Extra space |
|---|---|---|---|---|---|---|---|
| 1 | `bubble_sort` | yes | yes | O(n) | O(n²) | O(n²) | O(1) |
| 2 | `merge_sort` | no | yes | O(n log n) | O(n log n) | O(n log n) | O(n) |
| 3 | `quicksort` | yes | no | O(n log n) | O(n log n) | **O(n²)** | O(log n) |

`quicksort` publishes `pivot_policy: median_of_three` and
`known_limitations: ["all-duplicate input degrades to O(n^2)"]`. Both are mandatory, not
optional — see [ADR-003](/decisions/adr-003-pivot-policy.md) and
[the pivot trap](/algorithms/quicksort.md#the-pivot-trap).

Order matters and is declared, not incidental. Bubble sort runs first because it is the
cheapest to reason about and the slowest, which makes it the clearest contrast; quicksort runs
last because it is the one a caller is most likely to reach for.

## The contract

An algorithm must satisfy all of the following to be registered.

### Interface

```
sort(input: []int64) ([]int64, Metrics)
```

| Obligation | Requirement |
|---|---|
| Accept the array | Read-only in spirit, but see mutation below. |
| Return | The sorted array. Always the buffer it was given, never a substituted one. |
| Populate `Metrics` | `comparisons`, `moves`, and `auxiliary_slots`. |

### Hard rules

1. **Operate only on the buffer provided.** The orchestrator guarantees a fresh copy
   ([FR-4](/requirements/functional-requirements.md)); in-place implementations are expected
   to use it in place. An implementation that reaches for a shared global scratch buffer
   breaks the order-independence the orchestrator depends on.
2. **Return the same buffer, sorted.** Returning a new allocation is permitted only for
   out-of-place implementations, and even then the returned array is what gets verified.
3. **Reset counters at the start of every execution.** Counters must not accumulate across
   warm-up or iterations
   ([Rule 5](/design/step-counting-semantics.md#rule-5--counters-reset-per-execution)).
4. **Count exactly per the normative rules.** Including pivot-selection comparisons, and
   counting a swap as 2 moves. An implementation that counts differently **MUST** publish a
   distinct `step_counting_version` rather than silently reporting incomparable numbers.
5. **Declare `step_counting_version`.** Mandatory. This is what lets the registry expose two
   algorithms that count steps under different rulesets without the comparison being
   meaningless — the response must make the mismatch visible.
6. **Declare `pivot_policy` when applicable.** For partition-based algorithms this is not
   optional: worst-case complexity is a property of the algorithm *and* its pivot policy.
7. **Declare `known_limitations`.** An empty list must be a deliberate claim that no input
   defeats the algorithm, not a default.
8. **Never throw, panic, or loop forever on any valid input.** The orchestrator applies a
   time budget ([FR-11](/requirements/functional-requirements.md)), but a registry that only
   works on well-behaved inputs is not a teaching corpus — an algorithm that can hang on a
   specific input is itself a lesson, and the service should be able to report it.
9. **Handle n = 0 and n = 1** ([FR-2](/requirements/functional-requirements.md)). Zero
   comparisons, zero moves. No special-casing by the caller.

## Why algorithms are compiled in, not uploaded

The obvious extension — let callers submit an algorithm — is rejected permanently. It would
put user-supplied code inside the service's
[trust boundary](/architecture/system-context.md#trust-boundary), which means arbitrary code
execution with the service's privileges, reachable by an unauthenticated caller in phase 1
([security](/nfr/security.md)).

Everything an algorithm needs to do is expressible as a function from `[]int64` to
`[]int64` plus counters. There is no case for runtime code submission, and the cost of
getting it wrong is total compromise.

**Adding an algorithm is therefore a code change**: implement the interface, add counters per
the normative rules, register it with its declared properties, and add it to
[the golden-number tests](/roadmap/phase-1-sort-comparison.md). This is a real constraint and
[US-8](/requirements/user-stories.md) is scoped accordingly — "without touching the engine",
not "without deploying".

## Registry integrity

`GET /v1/algorithms` **MUST** fail loudly rather than emit an inconsistent registry. The
following are startup-time errors, not runtime warnings:

* Two algorithms sharing an `id`.
* An algorithm with an empty `id`, `name`, or `step_counting_version`.
* A quicksort-like algorithm with no `pivot_policy`.
* An algorithm whose declared `step_counting_version` is unknown to the service.

The service **SHOULD** also refuse to start if the compiled-in counting semantics do not
match the `step_counting_version` its instrumented algorithms declare. A registry that
validates at startup is what lets the HTTP layer trust it downstream and skip re-checking
per request.

## What the registry deliberately does not publish

* **A single "efficiency score".** See
  [ADR-001](/decisions/adr-001-metric-vector-over-scalar.md).
* **A recommended algorithm.** The honest answer depends on the input distribution, which the
  service does not know. Publishing "use quicksort" would be the
  [product brief's rejected non-story](/requirements/user-stories.md#non-stories).
* **Measured timing figures.** Timings from one machine are meaningless on another
  ([timing measurement](/design/timing-measurement.md)). Publishing them invites exactly the
  cross-environment comparison the service refuses to make.