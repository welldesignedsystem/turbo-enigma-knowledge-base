---
type: Roadmap
title: Future phases
description: Phase 2 and beyond — what is deferred, what unlocks it, and what each item is for.
tags: [roadmap, future, phasing]
status: draft
stale_after: 2027-01-02T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Future phases

Nothing here is committed. Each item states **what unlocks it**, so a proposal can be checked
against the reason it was deferred rather than treated as an arbitrary omission. The full
deferral table is in [scope and phasing](/requirements/scope-and-phasing.md).

## Phase 2 — Authenticated, queued, more algorithms

### Asynchronous execution

**Unlocked by:** raising the input limit, or sustained load making thread occupancy the
bottleneck, or a new workload with multi-second duration. See
[ADR-002 when to revisit](/decisions/adr-002-synchronous-execution.md#when-this-decision-should-be-revisited).

Migration is pre-planned: return `202` plus a status URL, keep the result document identical,
add `DELETE /v1/runs/{run_id}` for cancellation. Stored phase-1 runs stay interpretable because
`execution_mode` is already recorded
([ADR-002 migration path](/decisions/adr-002-synchronous-execution.md#migration-path)).

### Authentication and rate limiting

**Unlocked by:** any deployment beyond a trusted local network.

Adds authentication, ownership on runs (`403` where phase 1 returns `404`), and per-caller rate
limiting. Rate limiting currently cannot be per-caller because there are no identities
([NFR-S1](/nfr/scalability.md#nfr-s1--concurrency-bounded-by-cpu-work-not-by-memory)).

This also closes the phase-1 gap where **stored inputs are readable by anyone who can guess a
`run_id`** ([security](/nfr/security.md#what-phase-1-does-not-provide)).

### 3-way partition for quicksort

**Unlocked by:** nothing — it is a deliberate improvement, deferred only so the phase-1 cliff
stays visible to learners ([ADR-003](/decisions/adr-003-pivot-policy.md#the-limitation-this-decision-does-not-fix)).

Replaces Lomuto with a Dutch national flag partition, so equal keys are grouped rather than
pushed to one side. **This changes quicksort's measured counts on duplicate-heavy input**, so per
[NFR-M4](/nfr/maintainability.md#nfr-m4--registry-changes-do-not-rewrite-history) it requires a
version bump and must not retroactively alter archived results.

Worth noting the teaching tension: fixing this removes the best exercise in the trio. If it is
fixed, `known_limitations` should say the cliff *was* there, and the
[ADR-003 numbers](/algorithms/quicksort.md#the-pivot-trap) stay as the historical record.

### Run listing

**Unlocked by:** wanting to browse runs without remembering identifiers — the [US-7](/requirements/user-stories.md)
workflow only needs retrieval by ID.

Adds `GET /v1/runs` with pagination and filtering by `input_digest`. This is the point at which
the store stops being purely a key-lookup workload and [scanning questions](/nfr/scalability.md#what-is-deliberately-not-designed-for)
arrive.

### More algorithms

**Unlocked by:** wanting more contrast, not more coverage. The phase-1 trio already spans the
space ([product brief](/requirements/product-brief.md#the-first-example-sort-comparison)).

If added: heapsort (in-place, guaranteed O(n log n) — completes the comparison with merge sort
and quicksort), then insertion sort (shows a third best case, and is the practical winner on
small arrays, which is a genuinely surprising result worth reaching).

### Store restructuring

**Unlocked by:** storage growth outgrowing the primary store's working set.

Move `input` and `results[]` to object storage, keeping the `Run` shell plus pointers
([data model](/architecture/data-model.md#on-storing-input)), and define a retention policy — which
[no one has specified yet](/nfr/scalability.md#nfr-s3--store-growth-and-retention-unspecified).

## Phase 3 — Trace output, more input types, historical analysis

### Step-by-step trace

**Unlocked by:** learners wanting to see *why*, not just *how many*.

Returns per-algorithm state snapshots or operation events. Large responses, useful for a
debugger rather than an API. Needs a size limit and probably an asynchronous path, since a
10,000-element trace is enormous.

### Additional input types

**Unlocked by:** demand beyond integers.

Floats and strings require re-specifying
[FR-1's numeric contract](/requirements/functional-requirements.md#fr-1--accept-an-integer-array),
and raise a real semantic question: with floats, does the comparison test use `<` or `<=`?
Equal-but-distinct keys with no meaningful tie-break also interact with
[stability](/algorithms/quicksort.md#stability). Not a small change.

### Historical trending

**Unlocked by:** a defensible methodology first.

Deliberately last. Trending `elapsed_ns` is prohibited
([NFR-O3.6](/nfr/performance.md#nfr-p3--timing-measurement-methodology-bounded)) because the
methodology does not support it — trending noisy timings is worse than not trending them, because
it produces a performance claim with the authority of the service's own storage.

Trending **`comparisons`** is legitimate immediately, since step counts are portable. Distributions
of step counts by input shape would show clients shifting from sorted to adversarial inputs, which
[step counts detect and timings do not](/nfr/observability.md#nfr-o2--per-algorithm-observability).
A phase-2 item, not phase 3 — it needs no new methodology, only storage.

### Dedicated measurement workers

**Unlocked by:** an actual need to publish timings — which the
[product brief](/requirements/product-brief.md#what-it-explicitly-is-not) says there is not.

Would require pinned cores, fixed thread configuration, GC accounting, and recorded environment
metadata ([timing measurement](/design/timing-measurement.md#if-timings-ever-need-to-be-trustworthy)).
Even then the result describes that machine at that moment. Step counts remain the only portable
figures in the service.

## Explicitly never

* **Accepting user-uploaded algorithms.** Remote code execution in an unauthenticated,
  CPU-bound, network-exposed service. Not a phase — a permanent exclusion
  ([NFR-X2](/nfr/security.md#nfr-x2--no-remote-code-execution-surface)).
* **A composite performance score.** Actively harmful: it invites exactly the conflated
  reasoning the service exists to correct
  ([ADR-001](/decisions/adr-001-metric-vector-over-scalar.md),
  [non-stories](/requirements/user-stories.md#non-stories)).
* **Telling users which algorithm is fastest.** The answer depends on the input distribution,
  which the service does not know.

## Sequencing note

The two highest-value phase-2 items are not the same as the two most-requested-feeling ones.
Authentication matters more than more algorithms for anything beyond a local network; the 3-way
partition matters less than it appears, because the cliff it fixes is currently the most
instructive behaviour in the service.