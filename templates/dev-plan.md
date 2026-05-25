---
id: "{{YYYY-MM-DD-HHMM-<request-slug>}}"
status: "{{planned|executing|complete|blocked}}"
created: "{{YYYY-MM-DD HH:MM}}"
request: "{{short summary}}"
---

# Development Plan: {{request_title}}

## User Request

{{Verbatim user request or a precise restate that captures both the what and any explicit constraints.}}

## Architecture Read Set

Docs loaded before planning. (Don't pad — only what was actually read.)

| Document | Why read |
| --- | --- |
| `MANIFEST.md` | Locate candidate modules. |
| `COVERAGE.md` | Check coverage of candidate areas. |
| `RISKS.md` | Check known risks touching candidates. |
| `modules/M####-<slug>.md` | Target module. |
| `interactions/<map>.md` | Blast-radius reasoning. |

## Classification

Single-module / multi-module / bug fix / refactor / integration / test-only / docs-only / unknown.

## Target Modules

| Module | Role in this change | Confidence |
| --- | --- | --- |
| `M####-<slug>` | {{primary | adjacent}} | {{confidence}} |

## Allowed Edit Surface

The ONLY ranges that may be edited during execution.

| File | Lines / anchor | Module | Reason |
| --- | --- | --- | --- |
| `{{path}}` | {{lines or `funcName`}} | `M####-<slug>` | {{reason}} |

## Forbidden Nearby Areas

Areas that look related but must NOT be touched without self-healing first.

| File / area | Reason |
| --- | --- |
| `{{path or area}}` | {{why off-limits — e.g., "owned by M0042, change here would alter consensus invariant"}} |

## Expected Interaction Changes

How the global interaction graph will change.

| From | To | Change | Type |
| --- | --- | --- | --- |
| `M####` | `M####` | {{added/removed/signature changed/no change}} | {{call|event|data|...}} |

## Blast Radius

- {{One line: targeted only / parent-modules / system-wide. Justify.}}
- Verification scope set accordingly in the Verification Plan below.

## Self-Healing Triggers

Run self-healing if any of these appear during execution:

- {{Specific trigger for this plan — e.g., "any caller of `Pool.Add` not listed in M0007's Inbound."}}
- {{Specific trigger — e.g., "any new on-chain event not in interactions/data-flow.md."}}

## Implementation Steps

Ordered, small, each pointing at one allowed-surface entry.

1. {{Step — file:lines + what to change.}}
2. {{Step.}}
3. {{Step.}}

## Verification Plan

| Layer | Command / check | Expected result |
| --- | --- | --- |
| Targeted | `{{command — e.g., `go test ./consensus/pow/...`}}` | All pass |
| Affected modules | `{{command}}` | All pass |
| Broader (if blast radius warrants) | `{{command}}` | All pass |
| Build / type check | `{{command}}` | Clean |
| Lint | `{{command}}` | Clean |

## Rollback Notes

{{How to revert if the change must be undone. Specific commands or commit pattern. Note any state migrations that aren't trivially reversible.}}

## Open Risks

| Risk | Mitigation |
| --- | --- |
| {{risk}} | {{mitigation or "accepted, noted in RISKS.md as R####"}} |

## Heal Events

Appended during execution when self-healing fires. One entry per heal.

### {{YYYY-MM-DD HH:MM}} — {{trigger}}

- What was missing: {{...}}
- Wiki updates: {{paths}}
- New allowed surface: {{ranges}}
- See: `self-healing/YYYY-MM-DD-HHMM-<trigger-slug>.md`

## Execution Log

Brief running record.

| Time | Action | Result |
| --- | --- | --- |
| {{HH:MM}} | {{action}} | {{result}} |

## Final Report

Filled in at completion (mirrors what goes to the user in chat, plus persistence).

- Changed modules: `M####`, `M####`
- Wiki updates: {{paths}}
- Self-heals: {{count, link each report}}
- Verification: {{summary}}
- Remaining risks: {{summary}}
