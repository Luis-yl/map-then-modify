# Architecture Coverage

Last updated: `{{YYYY-MM-DD HH:MM}}`
Last verified at commit: `{{commit_sha}}`

## Summary

| Status | File count | LOC | Notes |
| --- | ---: | ---: | --- |
| Covered | {{count}} | {{loc}} | {{notes}} |
| Partially covered | {{count}} | {{loc}} | {{notes}} |
| Unknown | {{count}} | {{loc}} | {{notes}} |
| Excluded | {{count}} | {{loc}} | {{notes}} |

## File Coverage

One row per first-party file. Sorted by status (unknown / partial first), then path.

| Path | Status | Owning module(s) | Covered ranges | Confidence | Notes |
| --- | --- | --- | --- | --- | --- |
| `{{path}}` | {{covered|partial|unknown|excluded}} | `{{M####, M####}}` | {{ranges or "full file"}} | {{confidence}} | {{notes}} |

## Partial Coverage Details

| Path | Covered (ranges) | Not covered (ranges) | Why it matters | Next action |
| --- | --- | --- | --- | --- |
| `{{path}}` | {{ranges}} | {{ranges}} | {{risk if left uncovered}} | {{action}} |

## Unknown Areas

| Path or area | Why unknown | Risk | Suggested analysis |
| --- | --- | --- | --- |
| `{{path_or_area}}` | {{reason — e.g., no obvious entry point, dynamic loading}} | {{risk}} | {{next step}} |

## Excluded Areas

| Path | Reason | Source of truth | Notes |
| --- | --- | --- | --- |
| `{{path}}` | {{generated|vendored|binary|build-artifact|third-party-copy}} | `{{generator_path_or_upstream_url}}` | {{notes}} |
