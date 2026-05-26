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

## Reverse Validation Summary

> Filled by Phase 6.2 reverse pass. Demonstrates that the orphan-discovery step actually ran and resolved its results — not an empty placeholder.

| Metric | Count | Notes |
| --- | ---: | --- |
| First-party files enumerated (after Phase 0 exclusions) | {{count}} | from `rg --files` minus excluded paths |
| Files cited by at least one `modules/M*.md` `code_map` | {{count}} | forward-classification covered count |
| Orphans found by reverse pass | {{count}} | files with no owner |
| Orphans → moved to `excluded` | {{count}} | with reason added to Excluded Areas |
| Orphans → resolved via self-healing into existing module | {{count}} | which modules absorbed them, see `self-healing/` log |
| Orphans → resolved via self-healing into NEW module | {{count}} | new M#### IDs assigned, see `MANIFEST.md` |
| Orphans → marked `unknown` + `RISKS.md` entry | {{count}} | files with `R####` ref |
| Un-triaged orphans (deferred to next frontier) | **{{count}}** | mapping is NOT complete if this is > 0 |

If the last row is > 0, list each one explicitly in `MANIFEST.md` Next Frontier with one bullet per file.
