# Design

The decisions that are not obvious from the code someone would write by reading the
[requirements](/requirements/functional-requirements.md). Each answers a question of the form
"why this and not the obvious alternative?"

* [Run execution model](/design/run-execution-model.md) — how a run actually executes.
* [Step-counting semantics](/design/step-counting-semantics.md) — **read this before
  touching any number in this bundle.**
* [Timing measurement](/design/timing-measurement.md) — why `elapsed_ns` cannot rank.
* [Algorithm registry](/design/algorithm-registry.md) — the extension contract.
* [Input validation](/design/input-validation.md) — limits and rejection rules.