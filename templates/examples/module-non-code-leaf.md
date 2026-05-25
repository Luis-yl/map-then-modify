---
id: "M####-<slug>"
title: "<Pure-document module title — what contract / manual / policy it owns>"
parent: "M####-<parent_id>"
status: "mapped"
leaf: true
confidence: "high"
last_verified: "YYYY-MM-DD"
last_verified_sha: "<git_sha>"
---

<!-- EXAMPLE — A "non-code leaf" module owns documents (contracts, manuals,
     policies, ABI specs, README-style guides) rather than executable code.
     Many projects have one: an agent contract, a contributor guide, an
     ops manual, a compliance policy, a vendored upstream license file.

     This example shows how to fill out templates/module.md for that
     shape: structural anchors instead of line ranges, the document
     itself as the "implementation", invariants quoted from the doc,
     enforceability gaps documented in Unknowns And Risks instead of
     downgrading `confidence`.

     Strip these EXAMPLE comments and {{placeholders}} when adapting. -->

# M####-<slug>: <Module title>

## Summary

{{One to three short paragraphs describing what contract / manual /
policy this module owns. Plain language. Cite the most important
external concept the document deals with (e.g., "the API rate-limit
policy for partner integrators", "the contributor PR review SLA",
"the upstream LICENSE provenance").}}

This module is **prose, not code**. But it is load-bearing — violating its
clauses causes real consequences (regressions in {{which behavior}}, legal
exposure on {{which boundary}}, etc.).

## Architecture Role

Sits outside the code graph. Read by {{who — every Agent at session start /
every new contributor / every external partner during onboarding / etc.}}.
Has the highest "blast radius per word changed" in its scope — flip a
sentence and every future actor's behavior shifts.

## Scope

### Owns

- `{{path/to/document-1.md}}` — {{one line: what this doc is}}
- `{{path/to/document-2.md}}` — {{one line}}
- `{{path/to/document-3.md}}` — {{one line — e.g., "thin pointer that
  references the main contract for non-supported agents"}}

### Does Not Own

- The behavior implementations the contract refers to (`M####-<implementer>`).
- The systems whose policy this documents (`M####-<system>`).

## Code Map

> Pure-document leaves use `Lines: full` with structural anchors
> (`## Section Header`, `top-level`, or quoted clause label) rather
> than line ranges. Documents drift by reflow, not line shift.

| Role | File | Lines | Anchor | Edit Guidance | Confidence |
| --- | --- | --- | --- | --- | --- |
| {{What this doc establishes — e.g., "primary contract"}} | `{{path/to/document-1.md}}` | full | `## {{Main Section Header}}` | requires-careful-context | high |
| {{What this doc establishes — e.g., "project guide"}} | `{{path/to/document-2.md}}` | full | top-level | requires-careful-context | high |
| {{What this doc establishes — e.g., "pointer for non-supported agents"}} | `{{path/to/document-3.md}}` | full | top-level | safe | high |

## Inbound Interactions

| From | Mechanism | Evidence | Notes |
| --- | --- | --- | --- |
| {{who consumes this — e.g., "every Agent session" / "every new contributor"}} | {{how — `@import` / `Read` / manual reading / onboarding email}} | `{{evidence — e.g., "CLAUDE.md line @<path>"}}` | {{when consumed — at session start / during PR review / etc.}} |

## Outbound Interactions

| To | Mechanism | Evidence | Notes |
| --- | --- | --- | --- |
| `M####-<implementer>` | naming reference | mentions {{`tool_name`/`endpoint_path`}} | semantic contract on names |
| `M####-<system>` | policy reference | mentions {{system's user-facing behavior}} | the doc's clauses constrain the system's UI / API shape |

## Public Surface

For document modules, the "public surface" is the **named entities**
(tool names, route names, clause labels, field names) the document
commits to. Renaming any of these breaks every consumer that relies
on the name.

| Item | Kind | Stability |
| --- | --- | --- |
| `{{tool_or_clause_name}}` | tool name / contract clause | stable |
| `{{field_name}}` | field reference | stable |

## State And Data Ownership

_(none — pure document; "state" exists only in how consumers act
after reading.)_

## Invariants

Invariants for a document module are the **clauses the document commits to**.
Quote them precisely. The aggregator in Phase 7 cross-references these
with `RISKS.md` if any clause is honor-system-enforced (see Unknowns
And Risks below).

- {{Clause 1 — quoted or precisely paraphrased from the document.}}
- {{Clause 2.}}
- {{Clause 3 — include any specific numeric thresholds, time windows,
  or magic strings the document commits to. These are the things future
  changes might inadvertently break.}}

## Failure Modes

| Failure | Cause | Handling | Caller-visible result | Evidence |
| --- | --- | --- | --- | --- |
| {{Consumer skips a required clause}} | {{forgetfulness / contract not read}} | {{user / reviewer must re-flag}} | {{regression in X}} | `{{doc-path}}:{{section anchor}}` |
| {{Consumer misinterprets a clause}} | {{ambiguous wording / cultural translation gap}} | {{contract explicitly forbids}} | {{disengagement / wrong behavior}} | `{{doc-path}}:{{section anchor}}` |

## Tests And Verification

_(none — the contract is enforced by reader self-discipline and human
review. There is no automated linter that flags violation. This is the
enforceability gap — recorded in Unknowns And Risks below.)_

## Modification Guide

### Safe changes
- Adding a new clause that **restricts** future behavior (tightens the contract).
- Adding a new example / phrasing variant.

### Requires self-healing first
- Renaming any item in the Public Surface — coordinate with consumer modules.
- Removing or weakening an existing clause — record reason in `decisions/`.

### Avoid
- Editing this file in passing while working on something else. Contract
  changes have universal blast radius for their consumer set.

## Leaf Certificate

- [x] File paths documented (section anchors instead of line ranges, by design).
- [x] Stable anchors documented (section headers).
- [x] Inbound interactions documented.
- [x] Outbound interactions documented.
- [x] State and side effects documented (none, by nature).
- [x] Invariants documented.
- [x] Failure modes documented.
- [x] Tests or verification commands documented (or absence recorded as risk).
- [x] Nearby non-owned code identified.
- [x] Remaining uncertainty recorded in Unknowns And Risks below.

## Gotchas

- {{Historical context — e.g., "Clause X exists because of an incident in
  date Y when consumers misinterpreted Z. Do not soften."}}
- {{Cultural context — e.g., "The doc is in language L; non-L readers
  need to honor the format pattern not just translate the wording."}}
- {{Anti-refactor — e.g., "Don't reorganize section ordering — multiple
  consumers cite specific section anchors."}}

## Unknowns And Risks

- {{The enforceability gap — e.g., "No automated check detects clause
  violation after the fact. Project relies on human review."}}
- {{Translation / interpretation drift if relevant.}}

## Evidence Log

| Claim | Evidence |
| --- | --- |
| {{This module owns these documents}} | `{{path/to/document-1.md}}`, `{{path/to/document-2.md}}` |
| {{Specific clause}} | `{{path/to/document-1.md}}` § "{{Section Header}}" |
| {{Consumer reads at session start}} | `{{evidence — e.g., import line in another doc}}` |

<!-- END OF MODULE DOC — do not remove. Used by the resume protocol to detect partial writes. A MODULE.md missing this trailing line is treated as interrupted-mid-write and must be rewritten from scratch. -->
