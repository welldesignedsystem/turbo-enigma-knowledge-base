# Architecture

Four C4-style views, each at a different zoom level. Read
[system context](/architecture/system-context.md) first; the
[request lifecycle](/architecture/request-lifecycle.md) is the one to read when debugging
a run.

* [System context](/architecture/system-context.md) — the service boundary and its actors.
* [Component view](/architecture/component-view.md) — the modules inside the service.
* [Request lifecycle](/architecture/request-lifecycle.md) — what happens on `POST /v1/runs`.
* [Data model](/architecture/data-model.md) — entities, fields, lifecycles.