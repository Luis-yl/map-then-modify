# Self-Healing Report: {{trigger_slug}}

Time: `{{YYYY-MM-DD HH:MM}}`
Active dev plan: `{{dev_plan_path}}` (or `analysis-run` path if triggered during mapping)
Trigger classification: {{new module | child module | interaction edge | generated artifact | test contract | config boundary | external boundary | stale range | misplaced responsibility}}

## Trigger

{{What was discovered and why it matters to the current task. Two to four sentences.}}

## Previously Missing Or Stale Wiki Area

| Area | Problem |
| --- | --- |
| `{{path or module ID}}` | {{missing | stale anchor | contradictory | wrong owner | broader public surface than docs admit}} |

## Analysis Performed

Evidence gathered to understand the area. This is the same evidence standard as mapping (`boundary-and-evidence-rules.md`).

| Evidence | Finding |
| --- | --- |
| `{{path}}:{{lines}}` ({{symbol}}) | {{finding}} |
| `{{command output excerpt}}` | {{finding}} |

## Wiki Updates

Every `.architecture/` file touched by this heal.

| File | Update |
| --- | --- |
| `modules/M####-<slug>.md` | {{created | updated section X | bumped last_verified}} |
| `interactions/<map>.md` | {{added edge | updated edge}} |
| `inventories/<file>.md` | {{added item}} |
| `COVERAGE.md` | {{file X moved from unknown → covered}} |
| `RISKS.md` | {{added R####}} |
| `MANIFEST.md` | {{added module to index, updated module tree}} |

## Dev Plan Updates

| Section | Change |
| --- | --- |
| Allowed Edit Surface | {{added ranges}} |
| Forbidden Nearby Areas | {{added}} |
| Self-Healing Triggers | {{added trigger to prevent re-occurrence?}} |
| Open Risks | {{added}} |

## New Allowed Edit Surface (added by this heal)

| File | Lines / anchor | Module | Reason |
| --- | --- | --- | --- |
| `{{path}}` | {{lines or anchor}} | `M####-<slug>` | {{reason}} |

## Remaining Risk

{{What is still uncertain after the heal. If nothing, say "none". If something, link to the `RISKS.md` entry.}}

## Continuation Decision

{{Why it is safe to continue, OR why the work is blocked. If blocked, state what would unblock (e.g., "blocked: cannot find generator for `abi/generated.go` — need maintainer input").}}
