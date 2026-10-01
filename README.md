---
type: Reference
title: About this knowledge base
description: How this bundle is organized, who it is for, and how to contribute to it without breaking it.
tags: [meta, okf, contributing]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# About this knowledge base

This repository is a **knowledge bundle**: a set of linked Markdown documents describing a
proposed HTTP API service. It contains no service code. Every claim about the service is a
*design intent*, not an observation of running software.

Start at the [root index](/index.md).

## Who it is for

* **A student** learning how sorting algorithms behave and how to measure them honestly.
  Read the [algorithm reference](/algorithms/complexity-reference.md), then the
  [step-counting semantics](/design/step-counting-semantics.md), then the
  [worked example](/api/worked-example.md).
* **An engineer** who will build the service. Read
  [functional requirements](/requirements/functional-requirements.md) →
  [architecture](/architecture/index.md) → [design](/design/index.md) →
  [API contract](/api/index.md), then the [ADRs](/decisions/index.md) to learn which
  choices are already settled and must not be silently reopened.
* **An AI agent** picking up work in this repo. See [AGENTS.md](/AGENTS.md).

## How the bundle is organised

```
index.md                    Entry point. Every other file is reachable from here.
log.md                      Reverse-chronological history of this bundle.
glossary.md                 Terms of art. "Step" is defined here and nowhere else.
requirements/               What the service must do, and in what order.
architecture/               How the pieces fit together. C4-style views.
design/                     The decisions that are not obvious from the code you would write.
api/                        The wire contract, with a real measured example.
nfr/                        Measurable quality targets, explicitly marked as unvalidated.
algorithms/                 Per-algorithm reference, including the pivot trap.
decisions/                  ADRs. Each records a settled choice and its rejected alternatives.
roadmap/                    Phase 1, then later phases.
risks/                      Known risks with mitigations.
references/                 Provenance for measured numbers.
```

## The one rule that keeps this bundle honest

**Distinguish design intent from measurement.** The bundle mixes two very different kinds
of statement, and conflating them is the main way documentation like this rots:

* *Design intent* — "the endpoint must return a `comparisons` count per algorithm."
  Unverifiable until code exists. Marked `status: draft`.
* *Measurement* — "bubble sort performed 15 comparisons on `[5,3,8,1,9,2]`."
  Produced by running instrumented reference implementations. The exact conditions are in
  [measurement provenance](/references/measurement-provenance.md).

Never promote a design intent to a measurement, or a measurement to a guarantee. The
measured numbers describe *the reference implementations described in this bundle*, not
any future production implementation, which may count steps differently unless it follows
[step-counting semantics](/design/step-counting-semantics.md).

## Trust and review

Every concept in this bundle currently carries:

```yaml
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
```

and **no** `verified` entry. Under [OKF §5.3](/decisions/adr-004-okf-as-bundle-format.md)
that places every concept at the **`unverified`** trust tier. That is accurate: an agent
drafted this bundle and no human has reviewed it.

When a human reviews a concept, record it. This is the only edit that raises a concept's
trust tier, and it is what stops the bundle from silently becoming machine-generated
rumour:

```yaml
verified: { by: human:<your-id>, at: 2026-10-05T09:00:00Z }
```

A concept verified by any `human:` actor moves to **`human-reviewed`**. A concept verified
only by a `process:` actor is **`machine-confirmed`**. Keep `generated` and `verified`
separate — who wrote a document is not the same question as who checked it.

## Conventions this bundle follows

* **Bundle-relative links** (`/api/error-model.md`), not `../`-relative ones, so links
  survive a file being moved. This is OKF's recommended form.
* **Every non-reserved `.md` file has YAML frontmatter with a non-empty `type`.** This is
  the only hard OKF conformance rule. `index.md` and `log.md` are reserved and must *not*
  carry frontmatter, except that the root `index.md` may declare `okf_version`.
* **Concept types are ours, not OKF's.** OKF explicitly declines to fix a type taxonomy.
  Types used here: `Product Brief`, `Requirements Document`, `User Story`,
  `Architecture Document`, `Design Document`, `API Endpoint`,
  `Non-Functional Requirement`, `Algorithm Profile`, `Decision Record`, `Roadmap`, `Risk`,
  `Reference`. A consumer must tolerate these being renamed.
* **Additive frontmatter is fine.** OKF forbids rejecting unknown keys, so extensions are
  cheap. Anything non-standard is called out in the body.
* **Relative dates are banned; absolute ISO 8601 with an offset is required**
  (`2026-10-02T00:00:00Z`). Relative dates rot silently in a knowledge base.
* **No file in this bundle is a specification of a running system.** Where the docs say
  "must", read "the service is intended to".

## Before you commit a change

1. New concept file? Add it to the relevant `index.md` and to the root
   [index.md](/index.md). An unindexed concept is effectively invisible.
2. Bump `generated.at` on every concept whose meaning changed. This is what lets a reader
   tell a fresh fact from a stale one.
3. If a design decision changed, write a new ADR in `/decisions` and mark the superseded
   one `status: deprecated`. Do not rewrite history in place.
4. If you measured something, record the conditions in
   [references/measurement-provenance.md](/references/measurement-provenance.md).
5. Add a `log.md` entry under today's date.
