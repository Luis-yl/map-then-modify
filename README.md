# map-then-modify

An **agent skill** for safely taking over a large, unfamiliar codebase (e.g. a complex blockchain project) and doing secondary development without accidentally breaking modules you didn't know existed.

Works with any agent platform that supports the [skills convention](https://github.com/anthropics/skills) — Claude Code, Codex CLI, Cursor, GitHub Copilot, Gemini CLI, OpenCode, Cline, and others. The protocol is platform-agnostic: it's a directory of `SKILL.md` + `references/` + `templates/` that any conformant agent loads and follows.

## What it does

**Two main modes + one auto mode:**

1. **MAPPING** — Deep recursive architecture analysis. Produces a `.architecture/` wiki: modules → submodules → leaves, with exact file/line/anchor boundaries, inter-module interactions, coverage classification, and explicit risk + confidence tracking.

2. **DEVELOPMENT** — User requests a change. The skill loads only the relevant wiki slices, writes an execution plan under `.architecture/dev-plans/`, then executes autonomously, strictly inside the wiki's documented boundaries.

3. **SELF-HEALING** (auto, runs inside either mode) — When reality diverges from the wiki (undocumented code surfaces, anchor stale, contradiction), the skill analyzes the divergent area, updates the wiki, expands the plan, and continues. The user is informed in 1–2 sentences but **not asked** for technical approval — they don't know the codebase.

## Why it exists

To defeat one specific failure mode: an Agent edits a file in a large codebase, silently breaks a contract in another module it didn't know existed, and the user (who also doesn't know the codebase) cannot catch the regression.

The fix is **map before you modify**, with:
- Stable module IDs that survive renames/splits/merges.
- Range-level + stable-anchor code locations (line numbers alone rot).
- Explicit `confidence` on every claim (`high|medium|low|unknown`).
- Coverage classification of every file (`covered|partial|excluded|unknown`).
- Strict boundary enforcement during edits.
- Auto-self-healing when the wiki is wrong.

## Install

Clone the repo into your agent's skills directory. The exact path varies by platform; pick whichever matches your setup:

| Platform | Skills directory |
|---|---|
| Claude Code | `~/.claude/skills/` |
| Codex CLI | `~/.codex/skills/` |
| Cursor / VS Code with Copilot | `.cursor/skills/` or `.copilot/skills/` per project |
| Gemini CLI | `~/.gemini/skills/` |
| OpenCode / Cline / others | follow each platform's skills-loader convention |

```bash
# Replace <skills-dir> with your platform's directory from the table above
mkdir -p <skills-dir>
git clone git@github.com:Luis-yl/map-then-modify.git <skills-dir>/map-then-modify
```

Or clone anywhere convenient and symlink:

```bash
ln -s /path/to/your/clone <skills-dir>/map-then-modify
```

Restart your agent session. The skill should appear in the available-skills list.

## Trigger

The skill auto-triggers when you say things like:

- "啃下这个项目"
- "分析这个仓库的架构"
- "二次开发"
- "我要改这个大项目但不懂"
- "map this codebase"
- "I want to modify X in this big project safely"
- "this repo is huge and I'm scared to touch it"
- "safe refactor in an unfamiliar repo"

You can also explicitly invoke it via the Skill picker.

## Output (in your project)

```
<repo>/.architecture/
├── README.md MANIFEST.md COVERAGE.md RISKS.md
├── modules/M{id}-{slug}.md                                  # one file per module
├── interactions/{runtime,data-flow,build-and-config,external-boundaries}.md
├── inventories/{entrypoints,tests,external-interfaces,generated-and-vendor}.md
├── analysis-runs/  dev-plans/  self-healing/  decisions/    # process records
└── .meta/{progress.json, preflight.json}                    # resume + metadata
```

Add `.architecture/` to git — it's the project's permanent architectural memory.

## Skill structure

```
SKILL.md                                  # entry point, mode dispatch, principles P1–P9
references/
  ├── analysis-protocol.md                # mapping: phases 0–7 + subagent strategy
  ├── development-protocol.md             # secondary dev: phases 0–9 + blast radius
  ├── boundary-and-evidence-rules.md      # evidence/confidence/edge types/IDs/staleness
  └── self-healing-protocol.md            # divergence procedure, quality bar
templates/                                # 11 templates for every artifact under .architecture/
```

## Status

Initial release. Built via Claude + codex red-team / blue-team review.

## License

MIT.
