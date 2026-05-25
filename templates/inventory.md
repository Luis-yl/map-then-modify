# Inventory: {{inventory_name — entrypoints | tests | external-interfaces | generated-and-vendor}}

Last updated: `{{YYYY-MM-DD HH:MM}}`

## Purpose

{{What this inventory tracks. One paragraph.}}

## Items

| Item | Type | Evidence | Related module(s) | Notes |
| --- | --- | --- | --- | --- |
| `{{item — e.g., command name, route path, generated file, dependency name}}` | {{type}} | `{{path}}:{{lines}}` | `{{M####-slug, ...}}` | {{notes}} |

## Observations

Cross-cutting observations that don't fit one row.

- {{Observation — e.g., "All HTTP routes register through one central router; new routes must call `Router.Add()` to be discoverable."}}

## Gaps

Things this inventory knows it doesn't fully cover.

| Gap | Risk | Next action |
| --- | --- | --- |
| {{gap}} | {{risk}} | {{action}} |
