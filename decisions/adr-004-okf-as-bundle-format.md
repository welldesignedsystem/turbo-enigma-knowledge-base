---
type: Decision Record
title: "ADR-004: OKF v0.2 as the knowledge bundle format"
description: This documentation repository is an Open Knowledge Format v0.2 bundle: Markdown with YAML frontmatter, reserved index.md and log.md, versioned trust metadata.
tags: [adr, okf, documentation, tooling]
status: accepted
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
sources:
  - id: okf-spec
    resource: https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md
    title: Open Knowledge Format specification v0.2
---

# ADR-004: OKF v0.2 as the knowledge bundle format

**Status:** accepted · **Date:** 2026-10-02 · **Deciders:** bundle author (agent-drafted,
unreviewed — see [trust and review](/README.md#trust-and-review))

## Context

This repository is documentation for a service that does not exist yet. It must be readable by
humans without tooling, by AI agents without a bespoke SDK, diffable in git, and portable between
tools and organisations.

It must also solve a problem specific to agent-maintained documentation: when most content is
machine-generated, a reader needs to know **where a claim came from**, **how much to trust it**,
and **whether it is still current**. Plain Markdown plus frontmatter makes none of those
answerable.

## Decision

Adopt **Open Knowledge Format v0.2** (OKF) — a directory of Markdown files with YAML frontmatter,
with `okf_version: "0.2"` declared in the bundle-root `index.md`.

### The conformance rules actually used

Only three are normative, and all three are satisfied throughout:

1. Every non-reserved `.md` file has parseable YAML frontmatter.
2. Every frontmatter block has a non-empty `type`.
3. `index.md` and `log.md` follow their reserved structures — in practice, **no frontmatter**,
   except that the root `index.md` may declare `okf_version`.

The trust, lifecycle, provenance, and computation families are optional. This bundle uses
`generated`, `status`, `stale_after`, and `sources`; it does not use
[Attested Computation](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md),
which is aimed at SQL-style derived values and does not fit a design document.

### The parts that carry weight here

| Field | Use in this bundle |
|---|---|
| `type` | Required. Values are **ours, not OKF's** — OKF explicitly declines to fix a type taxonomy. |
| `title`, `description`, `tags` | Index generation and filtering. |
| `generated: { by, at }` | Who last wrote the content and when. Required on every concept. |
| `verified` | **Absent on every concept today.** Its absence is meaningful. |
| `status: draft \| stable \| deprecated` | Every concept is `draft`; nothing has been reviewed. |
| `stale_after` | Set on measurement-bearing concepts, which are the ones that can rot. |
| `sources` | Attached where a concept rests on external material (complexity classes, the spec itself). |

## Why OKF rather than plain Markdown with conventions

The alternative — Markdown plus a house style guide — was rejected because a style guide cannot
express the one distinction this bundle most needs to make: **which content a human has checked
and which an agent produced.**

OKF makes trust a frontmatter field rather than a matter of convention, which means it is
queryable, and the bundle uses that immediately: **every concept here is `unverified`, because
none has been human-reviewed.** A reader can filter for it; an agent can check it before relying
on it; and the first human review of a concept produces a visible, recorded state change.

Without that, "an agent wrote all of this" and "a human reviewed all of this" look identical on
the page — and that is the specific failure mode OKF's `verified` field exists to prevent.

Secondary benefits: `stale_after` makes staleness a comparison instead of a judgement call, and
`step_counting_version` in the [data model](/architecture/data-model.md#run) is the same idea
applied to measurement.

## Why OKF rather than a documentation framework

| Rejected | Because |
|---|---|
| MkDocs / Docusaurus / Hugo | Require a toolchain to read a document. The stated requirement is that `cat` and `git clone` are sufficient — OKF's own framing. Adds build steps to a repo that has none. |
| A knowledge-management product (Notion, Confluence) | Puts content behind a platform, which is precisely the lock-in OKF exists to avoid. Not diffable. |
| A schema-validated domain format (JSON/YAML with a schema) | Better machine validation, worse human readability, and a hard dependency on a validator for a bundle that should be readable as-is. |
| Plain Markdown + conventions | Cannot express provenance or trust, which are the bundle's central concerns. |

## Costs, accepted

* **A frontmatter convention to maintain.** Every new file needs a `type` and a `generated`
  block. Cheap, and enforced by convention rather than by a tool in this repo.
* **An unproven-at-this-scale dependency on an evolving spec.** OKF is at v0.2 and its
  ecosystem — validators, `okfctl`, `okn` — is young. The bundle depends only on the three
  normative rules, which are unlikely to change in a breaking way; a v0.3 adding optional fields
  is additive by the spec's own versioning rule.
* **No bundled validator.** No CI exists in this repo
  ([AGENTS.md](/AGENTS.md#validation)), so conformance is checked by inspection. An
  `okfctl validate .` or `okn validate .` run is advisory and optional.

## Consequences

* Any agent can navigate the bundle by reading `index.md`, following
  [bundle-relative links](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md),
  and filtering on frontmatter.
* A future third-party OKF consumer would understand the bundle with no custom mapping.
* **Concept types are ours.** Renaming a type is a local decision, and a consumer must tolerate
  unknown types — which the spec requires and this bundle's
  [README](/README.md#conventions-this-bundle-follows) records.
* Trust tiers stay low until humans review content. That is accurate, not a defect.

## How to tell this decision was wrong

If OKF's normative rules changed incompatibly (a required field renamed, reserved filenames
changed), or if a consumer ecosystem failed to materialise such that plain Markdown with a style
guide would serve better. The mitigation for the former is cheap: the three conformance rules are
cheap to drop, and the frontmatter is ordinary YAML that a script could convert.

## Related

* [About this knowledge base](/README.md) — bundle conventions and the trust model.
* [OKF specification v0.2](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)
* [Measurement provenance](/references/measurement-provenance.md) — how OKF's provenance
  concepts are used for measured numbers.
* [AGENTS.md](/AGENTS.md) — the conventions an editing agent must follow.