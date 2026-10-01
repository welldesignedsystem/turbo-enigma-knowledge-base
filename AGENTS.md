---
type: Agent Instructions
title: AGENTS.md
description: Operating instructions for an AI agent editing this knowledge bundle. Read before making any change.
tags: [meta, agents, contributing]
status: draft
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# AGENTS.md

This repository is a **documentation-only knowledge bundle** in [OKF v0.2](/index.md)
format. It describes a proposed HTTP API (the "Algorithm Run Service") that sorts a
submitted integer array with several algorithms and returns comparative metrics.

**There is no application code in this repo, and you should not add any.** If a task
appears to require source code, it belongs in a different repository — say so rather than
creating a `src/` directory here.

## What to read first

- [README.md](/README.md) — bundle layout, trust model, and contribution conventions.
- [index.md](/index.md) — the full map. Start here for any orientation question.
- [glossary.md](/glossary.md) — **"step" is defined here and only here.** If your task
  touches step counting, timing, or complexity claims, read this before writing anything.

## Non-negotiable conventions

- **Every non-reserved `.md` file must have YAML frontmatter with a non-empty `type`.**
  `index.md` and `log.md` are reserved: they must have **no** frontmatter, except the
  bundle-root `index.md`, which may declare `okf_version: "0.2"`.
- **Use bundle-relative links** — `[text](/design/step-counting-semantics.md)`. Never
  `../`-relative links; they break when files move.
- **Every new concept must be linked from two places**: its own directory's `index.md`
  and the root `index.md`. Unindexed concepts are invisible.
- **Bump `generated.at`** on any concept whose meaning changed, to an absolute ISO 8601
  timestamp with offset. Never use relative dates like "last week" or "Q3".
- **Add a `log.md` entry** under today's `## YYYY-MM-DD` heading for every meaningful
  change. Newest first.
- **Do not add comments inside YAML frontmatter** explaining the fields. The spec is the
  reference; the bundle's own [README](/README.md#conventions-this-bundle-follows) records
  the local conventions.

## The distinction that matters most

This bundle mixes **design intent** ("the endpoint must return X") with **measurement**
("bubble sort did 15 comparisons on this input"). They are not interchangeable, and
collapsing them is the primary failure mode for agents editing these docs.

- Design intent lives under `status: draft` and is unvalidated until code exists.
- Measurement was produced by instrumented reference implementations whose exact
  conditions are recorded in
  [references/measurement-provenance.md](/references/measurement-provenance.md).
- **Measured numbers describe those reference implementations, not any future production
  implementation.** Only a build that follows
  [step-counting semantics](/design/step-counting-semantics.md) may claim those numbers.

If asked to "update the performance numbers" or "fix the complexity table", the correct
move is to change the *provenance* too, or to state clearly that the figure is now a
target rather than a measurement. Do not silently swap one for the other.

## Trust state

Every concept is currently at the **`unverified`** trust tier: `generated` is present but
no `verified` entry. Do not add a `verified:` entry unless a human actually reviewed the
concept — fabricating one is the worst possible edit to this repo, because it is the only
signal that distinguishes reviewed knowledge from model output. If a human reviews a
concept, add `verified: { by: human:<their-id>, at: <timestamp> }`.

## Settled decisions — do not silently reopen

Read the ADRs in [/decisions](/decisions/index.md) before proposing a design change. Three
are load-bearing and were chosen against plausible alternatives:

- [ADR-001](/decisions/adr-001-metric-vector-over-scalar.md) — the API returns a **metric
  vector** (`comparisons`, `moves`), not a single "number of steps". The scalar
  `work_units` that does exist is explicitly a convenience composite with an arbitrary
  1:1 weighting, and is not authoritative.
- [ADR-002](/decisions/adr-002-synchronous-execution.md) — phase 1 is **synchronous**.
  The async migration path is documented; do not design the phase 1 API as if it were async.
- [ADR-003](/decisions/adr-003-pivot-policy.md) — quicksort uses a **median-of-three**
  pivot with Lomuto partition. Note the documented, verified limitation: this does *not*
  fix the all-duplicate-keys case, which still degrades to O(n²).

To overturn one, write a **new** ADR and set the old one to `status: deprecated`. Do not
edit an accepted ADR to say the opposite of what it decided.

## Technical traps that are easy to reintroduce

If you write or review anything touching these, the bundle has verified specifics:

- **Input mutation.** Every algorithm must receive its own defensive copy. Bubble sort and
  quicksort sort in place; handing one algorithm the previous algorithm's output silently
  invalidates the whole comparison.
- **Wall-clock is not a ranking.** `elapsed_ns` is environment- and load-dependent. Only
  `comparisons` and `moves` are machine-independent. See
  [timing measurement](/design/timing-measurement.md).
- **Bubble sort bounds the request size limit,** not quicksort. At `n = 10,000` it performs
  49,995,000 comparisons. The input limit exists to protect bubble sort.
- **Step counting conventions are load-bearing.** A swap is 2 moves, not 1. Merge sort's
  copy into the auxiliary buffer counts as a move *and* the copy back counts again. These
  choices are in [step-counting semantics](/design/step-counting-semantics.md); changing
  them silently invalidates every number in the bundle.
- **Output is verified before returning.** Each algorithm's result is checked against a
  reference sort of the input; a mismatch fails the run rather than returning wrong data.

## Validation

No CI, build, or test tooling is configured in this repo. To check bundle conformance
after an edit, verify by inspection that:

1. Every non-reserved `.md` file parses as YAML frontmatter with a non-empty `type`.
2. No `index.md` or `log.md` has frontmatter (root `index.md`: only `okf_version`).
3. Every bundle-relative link target exists.
4. `generated.at` is absolute ISO 8601 with offset.

Optionally validate with third-party OKF tooling, e.g. `okfctl validate .` or
`okn validate .` if one is installed. Neither is vendored into this repo, so treat their
output as advisory.
