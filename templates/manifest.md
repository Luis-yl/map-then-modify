# Architecture Manifest

## Repository Snapshot

| Field | Value |
| --- | --- |
| Repository | `{{repository_name}}` |
| Root | `{{repository_root}}` |
| Branch | `{{branch}}` |
| Commit | `{{commit}}` |
| Last analyzed | `{{last_analyzed}}` |
| Primary languages | `{{languages}}` |
| Package managers | `{{package_managers}}` |
| Build systems | `{{build_systems}}` |
| LOC (first-party) | `{{loc_total}}` |

## Commands

| Purpose | Command | Status | Notes |
| --- | --- | --- | --- |
| Install | `{{install_command}}` | `{{verified|unverified|n/a}}` | {{notes}} |
| Build | `{{build_command}}` | `{{verified|unverified|n/a}}` | {{notes}} |
| Test | `{{test_command}}` | `{{verified|unverified|n/a}}` | {{notes}} |
| Lint/typecheck | `{{lint_or_typecheck_command}}` | `{{verified|unverified|n/a}}` | {{notes}} |
| Run/start | `{{run_command}}` | `{{verified|unverified|n/a}}` | {{notes}} |

## Module Tree

Indented by depth. Each line: `ID — title (status, confidence)`.

```
M0000-root — <repo purpose in one phrase>
  M0001-<slug> — <title> (mapped, high)
    M0010-<slug> — <title> (leaf, high)
    M0011-<slug> — <title> (leaf, medium)
  M0002-<slug> — <title> (hypothesis, low)
```

## Module Index

| Module ID | Title | Parent | Leaf | Confidence | File |
| --- | --- | --- | --- | --- | --- |
| `M0001-<slug>` | {{title}} | `M0000-root` | no | high | `modules/M0001-<slug>.md` |

## Concern Index

Cross-cutting view: which modules own which kind of concern. Useful for routing dev requests.

| Concern | Primary modules | Notes |
| --- | --- | --- |
| Runtime entrypoints | `M####, M####` | {{notes}} |
| Persistence / state | `M####` | {{notes}} |
| External APIs | `M####` | {{notes}} |
| Protocol / consensus / security | `M####` | {{notes}} |
| Build / deployment | `M####` | {{notes}} |
| Tests / fixtures | `M####` | {{notes}} |
| Crypto primitives | `M####` | {{notes}} |
| Configuration | `M####` | {{notes}} |

## Exclusions

| Path | Reason | Source of truth (if generated) |
| --- | --- | --- |
| `{{path}}` | {{generated|vendored|binary|build-artifact|third-party-copy}} | `{{generator_path_or_upstream}}` |

## Analysis Status

| Area | Status | Confidence | Last updated | Notes |
| --- | --- | --- | --- | --- |
| `{{area}}` | {{covered|partial|unknown}} | {{confidence}} | {{date}} | {{notes}} |

## Next Frontier

What to map next, ranked.

| Priority | Area | Reason | Suggested action |
| --- | --- | --- | --- |
| 1 | `{{area}}` | {{reason}} | {{action}} |
