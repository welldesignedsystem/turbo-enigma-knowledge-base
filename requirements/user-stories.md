---
type: User Story
title: User stories
description: US-1 to US-8, the intent behind the functional requirements.
tags: [requirements, user-stories]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# User stories

These explain *why* [FR-1 … FR-12](/requirements/functional-requirements.md) exist. When a
requirement and a story disagree, the story wins and the requirement is wrong.

## US-1 — Reproducibility over speed

> As a student, I want the same array to produce the same comparison counts every time I
> run it, so that I can trust the numbers enough to build an intuition about why one
> algorithm beats another.

**Why this is the primary story.** It is the reason the service exists. It is also why
timing is reported but demoted, and why `elapsed_ns` must not be used to rank. A student on
a slow laptop gets identical `comparisons` to a student on a fast one.

Satisfied by FR-5, FR-7, FR-8.

## US-2 — See the trade, not a winner

> As a student, I want to see that quicksort and merge sort win by different amounts on
> different kinds of input, so I stop treating "fastest algorithm" as a fixed fact.

Concretely: submit a sorted array and a reverse-sorted array and watch
[quicksort's comparison count change drastically](/algorithms/quicksort.md#the-pivot-trap)
while merge sort's barely moves. That single observation is worth more than the entire
complexity table.

Satisfied by FR-4, FR-5, and the [worked example](/api/worked-example.md).

## US-3 — Find the cliff

> As an instructor, I want a learner to be able to find the input that breaks an algorithm,
> so that I can teach worst cases as something they *find* rather than something they are
> told.

The service deliberately ships [quicksort](/algorithms/quicksort.md) with a pivot policy
that does not fix the all-duplicate case. That is a teaching affordance, not an oversight.
A learner submitting `[7,7,7,…]` gets 589 comparisons at n = 32 — more than bubble sort's
worst case — and can work out why.

Satisfied by FR-3, FR-5 and the registry's published `pivot_policy` (FR-9).

## US-4 — Know when a number is meaningless

> As a reader, I want the service to tell me which of its numbers are trustworthy and
> which are not, so that I do not quote an unreliable figure as fact.

This is why `elapsed_ns` ships with a warning, why `work_units` is marked advisory, why
`step_counting_version` is recorded, and why [measurement provenance](/references/measurement-provenance.md)
is part of the bundle rather than an afterthought.

Satisfied by FR-7, FR-8, and the [README's honesty rule](/README.md#the-one-rule-that-keeps-this-bundle-honest).

## US-5 — Trust the output

> As a caller, I want to know the returned array is actually sorted, so that I can use the
> result without re-verifying it myself.

The caller almost certainly does not want to verify. FR-6 does it unconditionally and fails
the run on mismatch, because a silently wrong sort is the worst failure this service can
have — it would corrupt the learner's conclusion, not just their data.

Satisfied by FR-6, FR-12.

## US-6 — Get a useful error, not a stack trace

> As an integrator, I want a malformed request to produce an error that tells me which
> field is wrong and what the limit is, so I can fix it without reading the source.

Error bodies name the offending field, the offending value, and the applicable limit. See
the [error model](/api/error-model.md).

Satisfied by FR-1, FR-2, FR-3.

## US-7 — Reproduce someone else's run

> As an instructor, I want to hand a student a `run_id` and have them retrieve the exact
> same numbers, so our discussion is about the algorithm and not about who ran it when.

Satisfied by FR-8 and
[the run retrieval endpoint](/api/run-retrieval-endpoint.md).

## US-8 — Add my own algorithm without touching the engine

> As a developer, I want to add a fourth algorithm by implementing one interface, so that
> the registry, the API, and the verification harness work for it automatically.

Satisfied by the [algorithm registry](/design/algorithm-registry.md) contract. Note this
is a compile-time extension, not runtime code submission — see
[the product brief's exclusions](/requirements/product-brief.md#what-it-explicitly-is-not).

---

## Non-stories

These were considered and rejected for phase 1. They are not "later", they are not wanted:

* **"Let me upload my algorithm code."** Turns a teaching tool into a remote code execution
  service. Out of scope permanently unless the threat model changes entirely.
* **"Give me a single score out of 10."** Encourages exactly the conflated reasoning this
  service exists to correct.
* **"Tell me which algorithm is fastest."** The honest answer is "on this input, on this
  machine, at this moment" — and the service refuses to compress that into a ranking.