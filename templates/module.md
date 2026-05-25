---
id: "M{{####}}-{{slug}}"
title: "{{Module title — domain-specific noun phrase}}"
parent: "{{parent_module_id_or_M0000-root}}"
status: "{{hypothesis|mapped|leaf|partial|split|merged|deprecated}}"
leaf: {{true|false}}
confidence: "{{high|medium|low|unknown}}"
last_verified: "{{YYYY-MM-DD}}"
last_verified_sha: "{{git_sha}}"
---

# M{{####}}-{{slug}}: {{Module title}}

## Summary

{{One to three short paragraphs. A reader who has never seen this codebase must understand what this module does after reading this section. Use plain language. Cite the most important external concept the module deals with (e.g., "the Cosmos SDK staking module", "ERC-4626 vault accounting").}}

## Architecture Role

{{How this module fits into the larger system. Two to four sentences. Where it sits in the request/block/transaction lifecycle, what it enables, what depends on it.}}

## Scope

### Owns

- {{Responsibility owned by this module.}}
- {{Another responsibility.}}

### Does Not Own

- {{Nearby responsibility owned elsewhere — say which module owns it: "Signature validation: owned by `M0012-crypto-verify`".}}
- {{Adjacent area that might look like this module's job but isn't.}}

## Code Map

Mandatory for leaves. Non-leaves may point at children instead.

| Role | File | Lines | Anchor | Edit Guidance | Confidence |
| --- | --- | --- | --- | --- | --- |
| {{What this range contributes}} | `{{path}}` | {{start}}-{{end}} | `{{symbol_or_anchor}}` | {{safe / requires-careful-context / requires-self-healing-first}} | {{confidence}} |

## Inbound Interactions

Who calls into this module.

| From | Mechanism | Evidence | Notes |
| --- | --- | --- | --- |
| `{{M####-slug or external}}` | {{call|event|data|config|network|test|runtime}} | `{{path}}:{{lines}}` | {{notes}} |

## Outbound Interactions

What this module calls or depends on.

| To | Mechanism | Evidence | Notes |
| --- | --- | --- | --- |
| `{{M####-slug or external}}` | {{call|event|data|config|network|storage|test|runtime|build}} | `{{path}}:{{lines}}` | {{notes}} |

## Public Surface

Functions / types / events / routes / ABI methods exposed to other modules. Anything not listed is internal.

| Symbol | Kind | Signature / shape | Stability |
| --- | --- | --- | --- |
| `{{symbol}}` | {{function|type|event|route|ABI}} | `{{signature}}` | {{stable|experimental|deprecated}} |

## State And Data Ownership

| Data / state | Owner | Access pattern | Evidence | Notes |
| --- | --- | --- | --- | --- |
| {{state name}} | `{{this module id}}` | {{read / write / emit / cache}} | `{{path}}:{{lines}}` | {{notes}} |

## Invariants

What must always be true. What this module promises callers / requires from callees.

- {{Invariant 1 — be precise. E.g., "Block height is monotonically increasing; height(N+1) = height(N) + 1."}}
- {{Invariant 2}}

## Failure Modes

| Failure | Cause | Handling | Caller-visible result | Evidence |
| --- | --- | --- | --- | --- |
| {{failure}} | {{cause}} | {{retry / abort / revert / panic / return error}} | {{visible result}} | `{{path}}:{{lines}}` |

## Tests And Verification

| Behavior | Test / command | Evidence | Notes |
| --- | --- | --- | --- |
| {{behavior}} | `{{command_or_test_name}}` | `{{path}}:{{lines}}` | {{coverage notes}} |

If coverage is thin or missing, state so explicitly here AND add a `RISKS.md` entry.

## Modification Guide

### Safe changes (local to this module)
- {{Type of change that touches only this module's owned ranges and preserves public surface.}}

### Requires self-healing first
- {{Type of change that may touch undocumented or external code.}}

### Avoid
- {{Type of change that belongs to another module — say which.}}

## Leaf Certificate

Complete only when `leaf: true`. Every item must be ticked.

- [ ] File paths and line ranges are documented.
- [ ] Stable anchors are documented for every range.
- [ ] Inbound interactions are documented with evidence.
- [ ] Outbound interactions are documented with evidence.
- [ ] State and side effects are documented.
- [ ] Invariants are documented.
- [ ] Failure modes are documented.
- [ ] Tests or verification commands are documented (or absence recorded as risk).
- [ ] Nearby non-owned code is identified.
- [ ] Remaining uncertainty is recorded in Unknowns And Risks below.

## Gotchas

Non-obvious things future Agents / humans must know. Mined from comments, git log, and your own read.

- {{Gotcha — e.g., "Hand-rolled crypto: do not refactor without consulting M0099-crypto-audit."}}
- {{Gotcha — e.g., "Off-by-one in version 0.4 retained for backwards compatibility; see commit abc123."}}

## Unknowns And Risks

- {{Concrete unknown or risk + suggested next action.}}

## Evidence Log

| Claim | Evidence |
| --- | --- |
| {{claim}} | `{{path}}:{{lines}}` (+ context: symbol / test / config / command output) |

<!-- END OF MODULE DOC — do not remove. Used by the resume protocol to detect partial writes. A MODULE.md missing this trailing line is treated as interrupted-mid-write and must be rewritten from scratch. -->

