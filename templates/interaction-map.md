# Interaction Map: {{map_name — runtime | data-flow | build-and-config | external-boundaries}}

Last updated: `{{YYYY-MM-DD HH:MM}}`

## Scope

{{What this map covers and what it deliberately excludes. One paragraph. E.g., "Runtime-time calls and lifecycle ownership between top-level modules. Excludes build-time dependencies (see build-and-config.md) and external network boundaries (see external-boundaries.md)."}}

## Edge List

| From | To | Type | Mechanism | Evidence | Confidence | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `{{M####-slug or external}}` | `{{M####-slug or external}}` | {{call|event|data|config|build|network|storage|test|runtime}} | {{how — e.g., "registers route", "publishes to channel", "writes to table X"}} | `{{path}}:{{lines}}` | {{confidence}} | {{notes}} |

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
