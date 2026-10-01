---
type: Non-Functional Requirement
title: NFR overview
description: Index of non-functional requirements, how targets are stated, and why none is yet validated.
tags: [nfr, overview, targets]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# NFR overview

## None of these targets has been measured

**No NFR in this bundle has a measured value, because the service does not exist**
([scope and phasing](/requirements/scope-and-phasing.md)). Every number below is a
**proposed target**, not an observation. They are marked `target` in frontmatter and should be
read as "what we intend to build to, to be validated in phase 1".

This is the same honesty rule the rest of the bundle follows
([the bundle README](/README.md#the-one-rule-that-keeps-this-bundle-honest)) applied to quality
attributes. An NFR quoted as a fact is indistinguishable from an SLO that was never met.

Moving a target from `target` to a measured value requires recording provenance, exactly as for
the [step-count figures](/references/measurement-provenance.md).

## Requirements

| ID | Requirement | Document | Target status |
|---|---|---|---|
| NFR-P1 | Worst-case run latency within budget | [performance](/nfr/performance.md) | target |
| NFR-P2 | Verification overhead bounded | [performance](/nfr/performance.md) | target |
| NFR-P3 | Timing measurement methodology bounded | [performance](/nfr/performance.md) | target |
| NFR-S1 | Concurrency bounded by CPU work, not by memory | [scalability](/nfr/scalability.md) | target |
| NFR-S2 | Horizontal scaling of the API tier | [scalability](/nfr/scalability.md) | target |
| NFR-S3 | Store growth and retention | [scalability](/nfr/scalability.md) | unspecified |
| NFR-R1 | No incorrect output, ever | [reliability](/nfr/reliability.md) | binding |
| NFR-R2 | Per-run time budget enforced | [reliability](/nfr/reliability.md) | target |
| NFR-R3 | Registry validated at startup | [reliability](/nfr/reliability.md) | binding |
| NFR-R4 | Stored runs immutable | [reliability](/nfr/reliability.md) | binding |
| NFR-R5 | Degradation over wrong answers | [reliability](/nfr/reliability.md) | binding |
| NFR-X1 | Untrusted input never reaches an algorithm | [security](/nfr/security.md) | binding |
| NFR-X2 | No remote code execution surface | [security](/nfr/security.md) | binding |
| NFR-X3 | Errors leak no internals or input data | [security](/nfr/security.md) | binding |
| NFR-X4 | Resource exhaustion bounded per request | [security](/nfr/security.md) | binding |
| NFR-O1 | Every request correlatable by `request_id` | [observability](/nfr/observability.md) | target |
| NFR-O2 | Per-algorithm timing observable | [observability](/nfr/observability.md) | target |
| NFR-O3 | Step counts not logged as performance telemetry | [observability](/nfr/observability.md) | binding |
| NFR-M1 | Counting semantics versioned and testable | [maintainability](/nfr/maintainability.md) | binding |
| NFR-M2 | Golden numbers guard against drift | [maintainability](/nfr/maintainability.md) | target |
| NFR-M3 | Adding an algorithm needs no engine change | [maintainability](/nfr/maintainability.md) | target |
| NFR-M4 | Registry changes do not rewrite history | [maintainability](/nfr/maintainability.md) | binding |

## Which requirements are binding versus targets

The distinction is deliberate and worth stating, because it changes how each is reviewed.

**Binding** requirements describe invariants that follow from the design and whose violation
would be a correctness or security defect. They are not negotiable and are not subject to
"close enough" — either the run failed verification or it did not. NFR-R1, R3, R4, R5, X1, X2,
X3, X4, O3, M1, M4 are binding.

**Targets** are numeric goals that depend on hardware, implementation language, and deployment
topology — none of which is decided yet. A target can be missed without the service being wrong.
NFR-P1, P2, P3, S1, S2, O1, O2, M2, M3 are targets.

The binding set is the one that survives contact with a real implementation, because those
invariants are what [the golden-number tests](/api/worked-example.md) and the phase-1
[definition of done](/roadmap/phase-1-sort-comparison.md) actually verify.

## What is missing, and honestly so

* **No technology is chosen.** No language, framework, or store appears in this bundle,
  because [this is a documentation-only repository](/README.md) and no code exists to inspect.
  NFRs that would otherwise be technology-specific are stated technology-neutrally, and the ones
  that genuinely depend on the choice are marked as such.
* **No capacity numbers.** Sizing requires request volume, which nobody has stated. NFR-S3 is
  explicitly `unspecified` rather than given a plausible-looking number.
* **No availability target.** An availability SLO is meaningless before a deployment topology
  exists.