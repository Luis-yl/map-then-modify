# Cross-Check Report Template

> The cross-check artifact is JSON (not markdown). This file documents the schema and is referenced from `references/analysis-protocol.md` Phase 4.2.

## File path

`.architecture/.meta/cross-checks/M####-<slug>.review.json`

One file per leaf module. Lives forever once written — it is the audit trail of "what the reviewer found independently from source".

## JSON schema

```json
{
  "module_id": "M####-<slug>",
  "reviewed_at": "<ISO8601 timestamp>",
  "reviewer": "<subagent identifier; 'main-agent' if not delegated>",
  "writer": "<subagent identifier or 'main-agent'; for trace>",
  "source_commit": "<git SHA the review was done against>",

  "claims": {
    "code_map": [
      {
        "file": "<repo-relative path>",
        "line_range": "<start-end or 'full'>",
        "anchor": "<symbol / route / config-key / section-header / test-name>",
        "role": "<one-line: what this range does within the module>"
      }
    ],
    "inbound": [
      {
        "from": "<M####-<slug> or backticked external label>",
        "mechanism": "<call | event_publish | event_subscribe | data | config | build | network | storage | test | runtime>",
        "evidence": "<file:line where the inbound edge is observable>"
      }
    ],
    "outbound": [
      {
        "to": "<M####-<slug> or backticked external label>",
        "mechanism": "<same enum>",
        "evidence": "<file:line>"
      }
    ],
    "public_surface": [
      {
        "symbol": "<exported name>",
        "kind": "<function | type | event | route | ABI | constant | other>",
        "evidence": "<file:line>"
      }
    ],
    "state_ownership": [
      {
        "data": "<state / table / file / cache name>",
        "access": "<read | write | emit | cache | own-and-mutate>",
        "evidence": "<file:line>"
      }
    ]
  },

  "notes": "<free-form observations the reviewer wants to flag but are not structural claims — e.g., 'this function looks like it duplicates M0019/foo, possible split candidate', or 'no test for the slashing path, R-candidate'.>",

  "reconcile": {
    "completed_at": "<ISO8601, written by orchestrator after Phase 4.3>",
    "agreement_rate": "<float in [0,1] across the 5 structural fields>",
    "per_field_diff": {
      "code_map": {"agreed": 0, "writer_only_kept": 0, "writer_only_dropped": 0, "reviewer_only_added": 0, "reviewer_only_rejected": 0},
      "inbound": { "...": "..." },
      "outbound": { "...": "..." },
      "public_surface": { "...": "..." },
      "state_ownership": { "...": "..." }
    },
    "writer_only_kept_examples": [
      {"field": "inbound", "claim": "<...>", "evidence_provided_in_re-verify": "<file:line>"}
    ],
    "reviewer_only_added_examples": [
      {"field": "outbound", "claim": "<...>", "evidence": "<file:line>"}
    ]
  }
}
```

## How the orchestrator uses it

1. **Phase 4.2**: orchestrator writes the `claims` and `notes` sections after dispatching the reviewer subagent.
2. **Phase 4.3**: orchestrator diffs claims against the writer's draft. Writes the `reconcile` section.
3. **Phase 7**: orchestrator scans all cross-check artifacts; modules with `agreement_rate < 0.7` get an `R####` entry in `RISKS.md` with `severity: medium` minimum (the gap between writer and reviewer is a real signal of mapping risk).
4. **Permanent record**: the file stays on disk as evidence of how thorough the mapping was. A future analysis run can compare new vs old agreement rates to track quality drift.

## What is NOT cross-checked

Interpretive prose:

- Summary
- Architecture Role
- Modification Guide (safe / requires self-healing / avoid)
- Gotchas
- Unknowns And Risks (these are aggregated to RISKS.md instead; agreement is not meaningful)

These reflect the writer's judgment about the module's role and norms. Two independent agents will phrase them differently; mechanical diff would produce false disagreement. The structural fields above are the only ones where independent re-derivation from source gives a meaningful agreement metric.
