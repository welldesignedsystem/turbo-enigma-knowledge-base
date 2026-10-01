---
type: Non-Functional Requirement
title: Security
description: Threat model, binding security requirements, and an explicit account of what phase 1 does not provide.
tags: [nfr, security, threat-model]
status: draft
stale_after: 2027-04-01T00:00:00Z
generated: { by: opencode/big-pickle, at: 2026-10-02T00:00:00Z }
---

# Security

Phase 1 has **no authentication and no authorisation**. That is a deliberate, scoped decision
with a known cost, documented below rather than left implicit. All requirements here are
**binding**.

## Threat model

Phase 1's realistic adversary is **any unauthenticated caller with network access**. There is no
privileged actor and no multi-tenancy. The assets worth protecting are the service's availability
and its integrity of output — not confidentiality of the input, which is the caller's own data.

| Asset | Threat | Control |
|---|---|---|
| Service availability | CPU exhaustion via large arrays | [NFR-X4](#nfr-x4--resource-exhaustion-bounded-per-request) |
| Output integrity | Wrong sort returned silently | [NFR-R1](/nfr/reliability.md#nfr-r1--no-incorrect-output-ever) |
| Process integrity | Malformed input reaching an algorithm | [NFR-X1](#nfr-x1--untrusted-input-never-reaches-an-algorithm) |
| Process integrity | Remote code execution | [NFR-X2](#nfr-x2--no-remote-code-execution-surface) |
| Information disclosure | Internals or input data in error bodies | [NFR-X3](#nfr-x3--errors-leak-no-internals-or-input-data) |
| Cross-tenant data | None — no tenants exist | n/a |

Note what is **not** in this table: data exfiltration, credential theft, privilege escalation.
None applies, because the service holds nothing belonging to anyone but its callers and exposes
no credential to steal.

## NFR-X1 — Untrusted input never reaches an algorithm

| Requirement | Statement |
|---|---|
| NFR-X1.1 | Input **MUST** be fully validated before any algorithm executes ([FR-1](/requirements/functional-requirements.md), [FR-2](/requirements/functional-requirements.md)). |
| NFR-X1.2 | The validator **MUST** be the only component that inspects raw request data ([component view](/architecture/component-view.md#dependency-direction)). |
| NFR-X1.3 | Non-integers, out-of-range values, and booleans **MUST** be rejected rather than coerced. |

X1.3 is a security requirement as well as a correctness one. Host languages differ in what they
coerce silently — a boolean becoming `1`, a float becoming a truncated integer, an oversized
literal becoming infinity — and each coercion removes a validation check that the rest of the
system relies on. See
[the input validation rationale](/design/input-validation.md#booleans-are-rejected).

NFR-X1.2 matters because validation drift is invisible: a new code path that reads `input`
without passing through the validator reintroduces the whole class of bugs at once.

## NFR-X2 — No remote code execution surface

| Requirement | Statement |
|---|---|
| NFR-X2.1 | Algorithms **MUST** be compiled into the binary. Runtime algorithm submission **MUST NOT** be implemented. |
| NFR-X2.2 | No request field may cause the service to load, import, evaluate, or interpret caller-supplied code or expressions. |

This rules out the most obvious "feature" the service could grow — upload your own algorithm to
compare — and it rules it out permanently
([registry rationale](/design/algorithm-registry.md#why-algorithms-are-compiled-in-not-uploaded)).
An unauthenticated arbitrary-code-execution path in a service that is CPU-bound and
network-exposed is total compromise, and there is no variant of it that is safe by
configuration.

Everything an algorithm needs is expressible as a function from `[]int64` to `[]int64` plus
counters.

## NFR-X3 — Errors leak no internals or input data

| Requirement | Statement |
|---|---|
| NFR-X3.1 | Error responses **MUST NOT** include stack traces, file paths, host-language error text, or registry internals. |
| NFR-X3.2 | Error responses **MUST NOT** echo the caller's input array or its contents. |
| NFR-X3.3 | Detail **MUST** be available server-side, correlated by `request_id`. |

**X3.2 is specific to this service** and is the requirement most likely to be implemented by
accident. A failure inside an algorithm can surface a host-language panic or bounds-error message
containing buffer contents — which would place the caller's input array into a response body and
potentially into an intermediary's logs. Input is the one thing this service handles that a
third party may be reading, so it must not appear in a body destined for someone else.

Error bodies are therefore built from the stable `code` plus a
[structured `details` map](/api/error-model.md), never from a raw exception. The `message` is a
fixed human-readable string per `code`.

## NFR-X4 — Resource exhaustion bounded per request

| Requirement | Statement |
|---|---|
| NFR-X4.1 | `max_input_length` **MUST** be enforced on every request. |
| NFR-X4.2 | An HTTP request body size cap **MUST** be enforced independently. |
| NFR-X4.3 | A per-run time budget **MUST** be enforced ([NFR-R2](/nfr/reliability.md#nfr-r2--per-run-time-budget-enforced)). |
| NFR-X4.4 | In-flight request count and queued bytes **MUST** be bounded. |

Because the algorithms run **in-process on the request thread**
([system context](/architecture/system-context.md#environmental-dependencies-that-shape-the-architecture)),
CPU cost is paid by the service itself. A single request of a few hundred thousand integers
would occupy a core for seconds — and this holds **with no attacker**, purely from a legitimate
caller sending an array larger than expected. The [input limit](/design/input-validation.md) is
therefore a security control as much as a performance one.

X4.2 is separate from X4.1 because they bound different things: the body cap bounds
*transport*, the length cap bounds *CPU*. A JSON array can be large in bytes while short in
elements, and vice versa.

## What phase 1 does not provide

Stated plainly, because a reader should not have to infer the gaps.

| Not provided | Consequence | Phase |
|---|---|---|
| Authentication | Anyone who can reach the service can run runs and read any `run_id` they can guess. | 2 |
| Authorisation | No concept of ownership; any `run_id` resolves for anyone ([`404` not `403`](/api/run-retrieval-endpoint.md#responses)). | 2 |
| Rate limiting per caller | Cannot be per-caller without an identity. Global limits only ([NFR-S1](/nfr/scalability.md#nfr-s1--concurrency-bounded-by-cpu-work-not-by-memory)). | 2 |
| Transport security | Depends on deployment; the service assumes TLS terminates upstream. | 2 |
| Audit logging | No actor to attribute an action to. | 2 |
| Secret management | No secrets in phase 1. | n/a |
| Input confidentiality | Inputs are stored ([data model](/architecture/data-model.md)). A stored run is as readable as its `run_id` is guessable. | 2 |

The most under-appreciated line is the last one. Run inputs are **persisted** so that
[US-7](/requirements/user-stories.md) works, which means the service holds a durable record of
whatever arrays callers submitted. With no authentication, `run_id` opacity is the only thing
standing between a caller and someone else's data. That is acceptable for a teaching tool with
no sensitive use, and it is exactly why it is written down rather than left implicit.

## Before this is exposed publicly

Deployment behind rate limiting at the edge, TLS, and monitoring of `verification_failed` and
`run_budget_exceeded` rates. A spike in either suggests either an implementation defect or an
attempt to find one, and both warrant a look.