# Interaction Map: {{map_name — runtime | data-flow | build-and-config | external-boundaries}}

Last updated: `{{YYYY-MM-DD HH:MM}}`

## Scope

{{What this map covers and what it deliberately excludes. One paragraph. E.g., "Runtime-time calls and lifecycle ownership between top-level modules. Excludes build-time dependencies (see build-and-config.md) and external network boundaries (see external-boundaries.md)."}}

## Edge List

> **From / To column conventions**:
> - A module ID: `M####-<slug>` — the most common case.
> - An external system label in backticks: `` `codex subprocess` ``, `` `OpenAI HTTP` ``, `` `SQLite (fs)` ``, `` `(fs data/foo/*.jsonl)` ``, `` `(external: stripe API)` ``. External labels SHOULD also appear as a row in `inventories/external-interfaces.md` so the edge has a discoverable anchor.
>
> **Type column** — use one of the 10 edge types from `references/boundary-and-evidence-rules.md` §8. The two event sub-types are distinct: `event_publish` (this module emits) and `event_subscribe` (this module listens). Use both directions explicitly when both endpoints are known.

| From | To | Type | Mechanism | Evidence | Confidence | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `{{M####-slug or external label}}` | `{{M####-slug or external label}}` | {{call \| event_publish \| event_subscribe \| data \| config \| build \| network \| storage \| test \| runtime}} | {{how — e.g., `publish('foo')`, `on('bar')`, `app.register(...)`, `bus.emit(...)`, `child_process.spawn`, `db.query(...)` }} | `{{path}}:{{lines}}` | {{confidence}} | {{notes}} |

## Critical Flows

Sequential walkthroughs of important lifecycles. One subsection per flow.

### {{Flow name — e.g., "Block validation"}}

1. `{{M####-or-external}}` {{action}}
2. `{{M####-or-external}}` {{action}}
3. `{{M####-or-external}}` {{action}}

Evidence:
- `{{path}}:{{lines}}` ({{symbol}})

Failure handling:
- {{What happens if step N fails — where the error propagates, what gets rolled back.}}

### {{Another flow}}

(repeat structure)

## Blast-Radius Notes

What types of change to which areas affect which downstream modules. Used by Development Mode to size verification.

| Change area | Potentially affected modules | Verification needed |
| --- | --- | --- |
| {{area — e.g., "public ABI shape of M0017"}} | `{{M####, M####, M####}}` | {{verification — e.g., "run integration suite + simulate one historical fork"}} |
