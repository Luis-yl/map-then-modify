# Analysis Run: {{scope — "initial" | "focused on M####" | "frontier sweep"}}

Started: `{{YYYY-MM-DD HH:MM}}`
Finished: `{{YYYY-MM-DD HH:MM}}`
Repository: `{{repository_name}}`
Branch: `{{branch}}`
Commit: `{{commit}}`
Run kind: {{initial | focused | resume | self-heal-promotion}}

## Goal

{{What this run set out to map / verify / refresh. One paragraph.}}

## Inputs Read

| Input | Purpose |
| --- | --- |
| `{{path_or_command}}` | {{purpose}} |

## Commands Run

| Command | Result | Notes |
| --- | --- | --- |
| `{{command}}` | {{passed|failed|not_run}} | {{notes}} |

## Modules Added Or Updated

| Module | Change | Confidence after |
| --- | --- | --- |
| `M####-<slug>` | {{created|updated|split|merged|deprecated}} | {{confidence}} |

## Coverage Delta

| Area | Before | After | Notes |
| --- | --- | --- | --- |
| {{area}} | {{covered|partial|unknown|excluded}} | {{covered|partial|unknown|excluded}} | {{notes}} |

## Key Findings

- {{Finding — non-obvious thing this run discovered.}}
- {{Finding.}}

## Risks Added Or Updated

| Risk ID | Location | Action |
| --- | --- | --- |
| `R####` | {{file or area}} | {{created|updated|resolved}} |

## Decisions Made

If the run made architectural judgment calls (split this differently than the obvious choice, named a module a specific way, classified something as excluded), link to the corresponding `decisions/` entry.

- `decisions/YYYY-MM-DD-HHMM-<slug>.md` — {{one-line decision summary}}

## Next Frontier

What this run leaves unmapped or partially mapped.

- {{Next analysis step + rationale.}}
