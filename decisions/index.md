# Architecture decision records

Each ADR records a **settled** decision, the alternatives that were rejected, and what would
make the decision wrong. Read these before proposing a design change — three of them are
load-bearing.

To overturn an accepted ADR, **write a new one** and mark the superseded ADR
`status: deprecated`. Do not edit an accepted ADR to say the opposite of what it decided
([NFR-M5.3](/nfr/maintainability.md#nfr-m5--documentation-changes-with-code)).

| ADR | Decision | Status |
|---|---|---|
| [ADR-001](/decisions/adr-001-metric-vector-over-scalar.md) | Return a metric vector, not a scalar step count | accepted |
| [ADR-002](/decisions/adr-002-synchronous-execution.md) | Synchronous execution in phase 1 | accepted |
| [ADR-003](/decisions/adr-003-pivot-policy.md) | Median-of-three pivot for quicksort | accepted |
| [ADR-004](/decisions/adr-004-okf-as-bundle-format.md) | OKF v0.2 as the bundle format | accepted |

None has been human-reviewed, so all four are at the `unverified` trust tier
([the bundle README](/README.md#trust-and-review)).