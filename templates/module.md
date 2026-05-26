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

<!-- VARIANT GUIDE — choose the right shape for this module:

  For LEAF modules (leaf: true):
    - Code Map: full 6-column schema (Role / File / Lines / Anchor / Edit Guidance / Confidence).
    - Public Surface: list every exported symbol with its signature shape.
    - Leaf Certificate: complete every checkbox.
    - State And Data Ownership, Invariants, Failure Modes: filled in detail.

  For NON-LEAF modules (leaf: false):
    - Code Map: simplified to 2 columns (Subtree | Child module ID), prefixed
      with the one-line note: "This is a non-leaf module. See child modules
      for line-range ownership." Children's docs hold the precise ranges.
    - Public Surface: may omit, OR list only the "facade" exports that
      aggregate children's surfaces. If children expose their own surfaces
      independently, omit.
    - Leaf Certificate: omit entirely.
    - State And Data Ownership: may delegate to children with a 1-line
      pointer ("Each child owns its own state — see children").
    - Invariants and Failure Modes: state module-WIDE invariants/failures
      here (that apply across all children), not child-specific ones.

  BOTH variants ALWAYS fill: frontmatter, Summary, Architecture Role, Scope
  (Owns / Does Not Own), Inbound Interactions, Outbound Interactions,
  Modification Guide, Gotchas (if any), Unknowns And Risks, Evidence Log,
  and the trailing END-OF-MODULE sentinel.

  Special case: PURE-DOCUMENT LEAF (a module that owns documents — contracts,
  manuals, policies — rather than executable code). See
  `templates/examples/module-non-code-leaf.md` for a worked example showing
  how to use structural anchors (`## Section Header`) and `Lines: full`
  instead of line ranges, and how to document the enforceability gap in
  Unknowns And Risks instead of downgrading `confidence`.
-->

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

Mandatory for leaves. Non-leaves use the 2-column variant (see VARIANT GUIDE above).

> **Lines column conventions**:
> - For code files (functions, classes, methods), use a precise line range like `42-118`.
> - For document files (`.md`, contracts, manuals, fixtures) where the whole file is the unit, set `Lines` to `full` and put a structural anchor (e.g., `## Section Header`, top-level, ABI name) in the Anchor column. Documents drift by reflow, not line shift — structural anchors survive.
> - If the Code Map row warrants a warning note (e.g., "verify-later", "drop-named", "potential drift"), ALSO add a parallel entry to `## Unknowns And Risks` below — RISKS.md aggregation only scans `## Unknowns And Risks`, not Code Map columns.

| Role | File | Lines | Anchor | Edit Guidance | Confidence |
| --- | --- | --- | --- | --- | --- |
| {{What this range contributes}} | `{{path}}` | {{start-end or `full`}} | `{{symbol/route/test/config-key/section anchor}}` | {{safe / requires-careful-context / requires-self-healing-first}} | {{confidence}} |

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

Non-obvious things future Agents / humans must know. Mined from `git log` on the relevant files, code comments, and your own reading. The goal: any reader landing on this module under stress should NOT be surprised. Include all of:

- **Historical context** — why does this code exist *in this exact shape*? Often the answer is "back-port self-heal from when X wasn't tracked yet" or "kept this odd structure because we tried Y and it broke Z". Future readers cannot mine `git log` for every module — pre-mine it here.
- **Anti-refactor reasons** — code that looks awkward but the awkward shape is load-bearing. Example: "this 200-line if-block is ugly but preserves the test-injection surface; refactor would require updating 30 test files."
- **Version-conditional behavior** — "v0.4 retained an off-by-one for backwards compat; see commit `abc123`."
- **Hand-rolled non-standard library use** — "hand-rolled crypto: do not refactor without consulting M####-crypto-audit."
- **"Do not touch X without Y" warnings** — load-bearing implicit contracts not enforced by the type system or tests.
- **Incident references** — code that exists or persists because of a past incident. Cite the incident date / commit / postmortem.

Examples (replace with your module's actual gotchas):

- {{Historical context — e.g., "`reconcilePromotedCounts()` exists because early data was missing the total_promoted column; it's a back-port self-heal."}}
- {{Anti-refactor — e.g., "The 230-line feature-flag block makes buildApp() hard to read. Future refactor candidate, but the in-app branching keeps the test-injection surface stable."}}
- {{Hand-rolled — e.g., "Hand-rolled BLS signature verify in `crypto/bls.go`; do not replace with library without protocol-team review."}}

## Unknowns And Risks

- {{Concrete unknown or risk + suggested next action.}}

## Evidence Log

| Claim | Evidence |
| --- | --- |
| {{claim}} | `{{path}}:{{lines}}` (+ context: symbol / test / config / command output) |

<!-- END OF MODULE DOC — do not remove. Used by the resume protocol to detect partial writes. A MODULE.md missing this trailing line is treated as interrupted-mid-write and must be rewritten from scratch.

LIFECYCLE: for LEAF modules, this sentinel is set only after Phase 4.3 reconciliation completes. During Phase 4.1/4.2 (writer + reviewer passes), the file ends with `<!-- DRAFT — pending cross-check -->` instead. The DRAFT sentinel is also treated as interrupted by the resume protocol (same recovery: rewrite from scratch).

NON-LEAF modules go directly from write to END-OF-MODULE — they do not have Phase 4.2/4.3 (they delegate structural fields to children). -->

