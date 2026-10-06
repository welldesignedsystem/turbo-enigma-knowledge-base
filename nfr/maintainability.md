---
type: Non-Functional Requirement
title: Maintainability
description: Requirements keeping the measurement semantics stable across code changes, and keeping algorithm addition cheap.
tags: [nfr, maintainability, testing, versioning]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Maintainability

This service has one overwhelmingly important maintenance property: **its numbers must remain
comparable across releases.** A correctness bug is a bad day; a silent change to what a
"comparison" counts makes every previously published figure wrong while every test still passes.

## NFR-M1 — Counting semantics are versioned and testable

| Requirement | Statement |
|---|---|
| NFR-M1.1 | The counting rules **MUST** exist as a single normative definition ([step-counting semantics](/design/step-counting-semantics.md)), not as per-algorithm code. |
| NFR-M1.2 | Every response **MUST** carry `step_counting_version` ([FR-8](/requirements/functional-requirements.md)). |
| NFR-M1.3 | Any change that would alter a reported number for a fixed input and implementation **MUST** bump the version. |
| NFR-M1.4 | Stored runs **MUST** retain the version they were produced under. |

M1.4 is why [retrieval never recomputes](/architecture/request-lifecycle.md#retrieval-path): a
run stored under `v1` keeps reporting `v1` figures after `v2` ships, so it remains interpretable
forever.

Without M1.2–M1.4, the corpus degrades exactly as published benchmark data usually does — numbers
quoted from an old release turn out to be incomparable to current ones, and nobody notices
because nothing on the surface changed.

## NFR-M2 — Golden numbers guard against drift

| Requirement | Statement |
|---|---|
| NFR-M2.1 | Tests **MUST** assert the [golden step counts](/api/worked-example.md) for `[5,3,8,1,9,2]`: bubble `15`/`16`, merge `10`/`32`, quicksort `8`/`30`. |
| NFR-M2.2 | Tests **MUST** assert bubble sort's worst case of exactly `n(n−1)/2` at n = 4, 8, 16, 32. |
| NFR-M2.3 | Tests **MUST** assert quicksort's all-duplicate cliff: **589** comparisons and **1,116** moves at n = 32. |
| NFR-M2.4 | Tests **MUST** assert that running all algorithms together yields the same per-algorithm metrics as running each in isolation. |
| NFR-M2.5 | A test **MUST** inject a deliberately broken algorithm and assert the run **fails** rather than returning wrong data. |

M2.4 is the regression test for [FR-4](/requirements/functional-requirements.md) and the
[defensive-copy rule](/design/run-execution-model.md#decision-1--one-defensive-copy-per-execution).
It is the test that catches the silent-buffer-reuse bug described there, which produces entirely
plausible numbers.

M2.5 is the test for [NFR-R1](/nfr/reliability.md#nfr-r1--no-incorrect-output-ever). A
verification gate that has never been observed rejecting a fault is not known to work.

**Quicksort figures must come from instrumented runs, not hand-derived expectations.** A manual
trace of quicksort does not reliably reconcile with an implementation
([documented](/api/worked-example.md#why-this-section-used-to-be-wrong-and-what-replaced-it)).
M2.1–M2.3 are regression fixtures against recorded output, not proofs.

## NFR-M3 — Adding an algorithm requires no engine change

| Requirement | Statement |
|---|---|
| NFR-M3.1 | A new algorithm **MUST** require only an implementation plus a registry entry. |
| NFR-M3.2 | The registry **MUST** be the single source for default selection, the discovery endpoint, and execution order ([component view](/architecture/component-view.md)). |
| NFR-M3.3 | Adding an algorithm **MUST NOT** require modifying the orchestrator, the validator, or the HTTP layer. |

NFR-M3.2 keeps the three consumers of the registry from drifting into three separate lists —
which would show up as an API advertising an algorithm it will not run.

[US-8](/requirements/user-stories.md) is scoped to this, with the honest caveat that "without
touching the engine" still means a code change and a deployment: algorithms are
[compiled in](/design/algorithm-registry.md#why-algorithms-are-compiled-in-not-uploaded).

## NFR-M4 — Registry changes do not rewrite history

| Requirement | Statement |
|---|---|
| NFR-M4.1 | Changing an algorithm's behaviour (pivot policy, partition scheme, counting rules) **MUST** change its published `step_counting_version` or `pivot_policy`. |
| NFR-M4.2 | `Result` **MUST** carry its own copy of the behavioural flags it depended on. |
| NFR-M4.3 | Input digest canonicalisation changes **MUST NOT** be applied retroactively to stored digests. |

The [duplicate-key fix planned for phase 2](/roadmap/future-phases.md) is the test case for all
three. Adding a 3-way partition changes quicksort's measured counts on duplicate-heavy input, so
without M4.1–M4.3 the fix would silently invalidate every archived quicksort result — and the
numbers would look perfectly reasonable while describing an algorithm that never produced them.

M4.3 is the same class of problem on the digest side: `input_digest` is a grouping key, and a
grouping key that changes meaning retroactively has silently broken every historical comparison
that relied on it ([data model](/architecture/data-model.md#on-input_digest)).

## NFR-M5 — Documentation changes with code

| Requirement | Statement |
|---|---|
| NFR-M5.1 | A behaviour change and its documentation change **MUST** land together. |
| NFR-M5.2 | An architectural change **MUST** be recorded as a new ADR; accepted ADRs **MUST NOT** be edited to say the opposite of what they decided. |
| NFR-M5.3 | Superseded ADRs **MUST** be marked `status: deprecated`, not deleted. |

M5.3 matters because a deleted ADR is indistinguishable from one that was never written. The
record of *why* a decision was made, and what was rejected, is often more valuable than the
decision — and it is the first thing an agent or engineer will want when the decision is
questioned.

## Change checklist

Any change to this service should be able to answer:

1. Does it alter a reported step count for a fixed input? → bump `step_counting_version`,
   update the golden numbers and [measurement provenance](/references/measurement-provenance.md).
2. Does it change a registry property or an algorithm's behaviour? → bump the published version,
   add a `log.md` entry, do not touch stored runs.
3. Does it change the API surface? → update the [contract](/api/index.md) and the
   [error codes](/api/error-model.md) together.
4. Does it change an architectural boundary? → new ADR, deprecate the old one.
5. Does it change what is trustworthy? → update the disclaimers in the same change.