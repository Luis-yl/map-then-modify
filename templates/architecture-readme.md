# Architecture Wiki

Repository: `{{repository_name}}`
Root: `{{repository_root}}`
Last full analysis: `{{last_full_analysis}}` (commit `{{last_full_analysis_sha}}`)
Last focused update: `{{last_focused_update}}`

## Purpose

This directory is the architecture control plane for safe development in this repository.

Agents (and humans) must consult it before changing code. If related code is missing from this wiki, update the wiki first, then continue development. This is non-negotiable — see `SKILL.md` principle P4.

## Safety Contract

- Code edits must stay inside documented module ranges or self-healed ranges.
- Line ranges must be refreshed (re-grep the anchor) before editing.
- Generated and vendored code must not be edited unless the wiki documents it as source.
- Unknown related code must be analyzed and added to this wiki before modification.
- Risks must be recorded in `RISKS.md` instead of hidden.
- Module IDs (`M0001-...`) are stable and never renumbered or reused.

## Navigation

| File / dir | Purpose |
|---|---|
| `MANIFEST.md` | Repository snapshot, commands, module tree, module index, exclusions, analysis status, next frontier. |
| `COVERAGE.md` | Per-file coverage classification (covered / partial / excluded / unknown). |
| `RISKS.md` | Active risks, verification gaps, stale docs, external system uncertainties. |
| `modules/` | One file per module (leaf and non-leaf). Filename: `M{id}-<slug>.md`. |
| `interactions/` | Cross-module edges and system flows (runtime / data-flow / build / external). |
| `inventories/` | Entrypoints, tests, external interfaces, generated and vendored code. |
| `dev-plans/` | Execution plan for every development request. |
| `self-healing/` | Reports of wiki updates triggered during work. |
| `analysis-runs/` | Records of analysis sessions (initial + focused). |
| `decisions/` | Architectural decisions made during mapping or development (ADR-style). |
| `.meta/preflight.json` | Structured repo snapshot used at analysis start. |
| `.meta/progress.json` | Resume state for in-progress analysis. |

## Development Workflow (quick reference)

1. Read `MANIFEST.md`, `COVERAGE.md`, `RISKS.md`, and relevant module docs.
2. Write a dev plan under `dev-plans/`.
3. Edit only documented ranges.
4. If undocumented related code appears, run self-healing first.
5. Run targeted verification, then broader based on blast radius.
6. Update this wiki for any module behavior or interaction change.

## Current Status

{{current_status_summary — 3–5 lines: mapping completeness, top open risks, last activity}}
