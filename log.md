# Directory Update Log

## 2026-10-02
* **Correction**: Every quicksort *comparison* figure in the bundle was wrong and has been
  re-derived. Worked example 8 → **17**; at n = 32, 103/126/181/496 → **151/177/265/589**. The
  `496` figures were valid only for a naive first-pivot quicksort; the two policies had been
  conflated. Root cause: the original harness omitted pivot-selection comparisons — the exact
  error [Rule 1](/design/step-counting-semantics.md#rule-1--comparisons) exists to prevent —
  and did not sort correctly either. All **move** figures were already correct and are
  unchanged, as are all of bubble sort's. Full account in
  [the correction record](/references/measurement-provenance.md#a-correction-to-these-figures).
* **Correction**: Reference implementations were rewritten behind a correctness gate — each must
  reproduce the reference sort at every length from 0 to 44 before any figure is taken. This is
  recorded because the defective harness reported entirely plausible step counts for wrong output.
* **Correction**: Merge sort's `comparisons` were presented as input-independent. They are not:
  80 on sorted, reverse-sorted and all-identical input at n = 32, ≈122 on random input, 129
  maximum. Only its `moves` are input-independent. New
  [Consequence 6](/design/step-counting-semantics.md#consequence-6--input-independent-is-true-of-merge-sorts-moves-and-false-of-its-comparisons).
* **Decision**: Added the normative rule that **a self-swap counts 2 moves** (Convention A), which
  Rule 2 previously left open. The alternative — self-swaps are 0 moves — is defensible and would
  change quicksort's figures substantially; it is documented as rejected, not silently dropped.
* **Correction**: The quicksort hand-trace in [the worked example](/api/worked-example.md) did not
  reconcile and was kept as a "negative result" demonstrating that quicksort cannot be hand-checked.
  The measurement was at fault, not the method. The trace is rewritten and reconciles exactly.
* **Correction**: Bubble sort's worked example claimed 15 comparisons was "one short" of the worst
  case. `5+4+3+2+1 = 15 = n(n−1)/2` exactly; the slack is in moves (16 against 30 for
  reverse-sorted).
* **Correction**: Empty input is a valid run per FR-2, but FR-1, the input-validation rule table,
  the request-lifecycle diagram and the endpoint parameter table all rejected it. Aligned to FR-2.
* **Correction**: `data-model.md` claimed all three phase-1 algorithms are stable; quicksort is
  not, and the `AlgorithmRegistration` field is now `stable` (boolean) to match every response
  example rather than `stability` (enum).
* **Correction**: ADR-003's claim that median-of-three costs "roughly 3%" of comparisons is wrong
  by an order of magnitude: 3 per partition over 31 partitions is **93 of 151** on sorted input at
  n = 32. ADR-003 itself stands.
* **Added**: [roadmap](/roadmap/index.md) — phase 1 build order, golden numbers, required
  non-functional tests, exit criteria, and deferred work.
* **Added**: [risk register](/risks/risk-register.md) — ten scored risks with mitigations and
  warning signals, plus the open questions that must be answered before implementation.
* **Fixed**: 9 broken heading anchors and 1 broken reference; all 744 bundle-relative links and
  all frontmatter now validate. `AGENTS.md` gained the frontmatter the conventions require.
* **Creation**: Established the bundle described by the [root index](/index.md) for the
  Algorithm Run Service documentation set: requirements, architecture, design, API
  contract, NFRs, algorithm reference, ADRs, roadmap, and risk register.
* **Initialization**: Adopted [OKF v0.2](/decisions/adr-004-okf-as-bundle-format.md) as
  the bundle format and declared `okf_version: "0.2"` at the bundle root.
* **Initialization**: Recorded measured comparison and step-count figures in
  [the worked example](/api/worked-example.md) and
  [the complexity reference](/algorithms/complexity-reference.md). Provenance for those
  figures is in [measurement provenance](/references/measurement-provenance.md).
* **Open**: No concept in this bundle carries a `human:` `verified` entry yet, so every
  concept is at the `unverified` trust tier. See [trust and review](/README.md#trust-and-review).
* **Open**: The corrected figures are agent-derived and **not yet human-reviewed**. The
  correction record above exists precisely because a plausible-looking figure was wrong; a
  reviewer should re-derive or spot-check them before any of this is treated as settled.
