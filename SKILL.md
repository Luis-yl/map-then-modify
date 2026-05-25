---
name: map-then-modify
description: |
  Take over an unfamiliar, large, or high-risk codebase (e.g. a complex blockchain project) and
  do safe secondary development. Two main modes plus a self-healing mode:
  (1) MAPPING — deep recursive architecture analysis that produces a wiki at .architecture/
  (modules → submodules → leaves, with exact file/line/anchor boundaries, interactions, coverage,
  and explicit risk + confidence tracking).
  (2) DEVELOPMENT — make a user-requested change strictly inside the wiki's boundaries, write an
  execution plan, then execute autonomously.
  (3) SELF-HEALING (auto, inside either mode) — when reality diverges from the wiki, analyze the
  divergent area, update the wiki, expand the plan, continue. Inform the user; do not interrogate.
  Trigger when the user says things like: "啃下这个项目", "分析这个仓库的架构", "二次开发",
  "我要改这个大项目但不懂", "map this codebase", "I want to modify X in this big project safely",
  "this repo is huge and I'm scared to touch it", "safe refactor in an unfamiliar repo".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
  - TaskCreate
  - TaskUpdate
  - TaskList
  - AskUserQuestion
---

# Map Then Modify

Treat `.architecture/` as the **control plane** for working in an unknown codebase. Do not edit code merely because the code is visible. Edit only after the relevant module's boundary, owned ranges, interactions, and verification path are documented. If work reveals related code missing from the wiki, **update the wiki first, then continue**.

This skill exists to defeat one specific failure mode: an Agent edits a file in a large codebase, accidentally breaks a contract in another module it didn't know existed, and the user (who also doesn't know the codebase) cannot catch the regression. The fix is **map before you modify**, with rigorous boundary enforcement, evidence requirements, and self-healing when the map is incomplete.

---

## Mode auto-detection (run this first, every invocation)

```bash
PROJECT_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
ARCH_DIR="$PROJECT_ROOT/.architecture"
# Mapping is "complete" only when ALL terminal artifacts exist — these
# are written together at the end of Phase 7. progress.json is NOT a
# reliable signal during execution (it is updated only at completion).
if [ -f "$ARCH_DIR/MANIFEST.md" ] && [ -f "$ARCH_DIR/RISKS.md" ] && [ -f "$ARCH_DIR/COVERAGE.md" ] && [ -f "$ARCH_DIR/README.md" ]; then
  echo "MODE=develop"
  echo "STATE=ready"
elif [ -f "$ARCH_DIR/MANIFEST.md" ]; then
  echo "MODE=mapping"
  echo "STATE=resuming"
else
  echo "MODE=mapping"
  echo "STATE=fresh"
fi
echo "ARCH_DIR=$ARCH_DIR"
```

- **`MODE=mapping`** + **`STATE=fresh`** → load `references/analysis-protocol.md` + `references/boundary-and-evidence-rules.md`. Build the wiki from scratch.
- **`MODE=mapping`** + **`STATE=resuming`** → load the same references. Use the filesystem-based resume protocol in `analysis-protocol.md` (scan existing `modules/*.md`, cross-reference `MANIFEST.md`, find gaps, validate sentinel on each existing file). **Do not rely on `progress.json`** — it is a completion snapshot, not a live tracker.
- **`MODE=develop`** + **`STATE=ready`** → load `references/development-protocol.md` + `references/boundary-and-evidence-rules.md`. Use the wiki to guide the change.

When self-healing triggers inside either mode, additionally load `references/self-healing-protocol.md`.

If the user explicitly asks to "re-analyze" or "rebuild the wiki", treat as `MODE=mapping` even if `.architecture/` exists. Back up the old one to `.architecture.bak-<timestamp>/` first; never silently overwrite.

---

## Core principles (apply in EVERY mode — non-negotiable)

### P1. Map before you modify
Never edit a leaf module before it has a `.architecture/modules/M{id}-{slug}.md` covering it with code map, interactions, state, invariants, failure modes, and tests. If the doc is missing for the area you must touch, **stop and self-heal first**.

### P2. Strict boundary enforcement
Every leaf module has explicit `code_map` entries: `{file, line_range, anchor, role}`. Edits may only touch lines inside those ranges of modules listed in the active dev plan. Anything else requires re-planning or self-healing. See `references/boundary-and-evidence-rules.md`.

### P3. Evidence required
Every architectural claim must be traceable to: a file:line range, a symbol, a route registration, a config key, a test, a manifest entry, or runtime command output. "Probably handles X" is not allowed — mark it `confidence: low` or `unknown` and record it as a risk.

### P4. Self-heal silently, inform briefly
When divergence is found (code outside the documented boundary, unlisted caller/callee, stale range, contradiction), **do not ask the user**. They do not know this codebase. Run the self-healing protocol, update the wiki, expand the plan, continue. Tell the user in 1–2 sentences what happened.

### P5. User confirmation policy
Ask the user **only** for decisions that cannot be inferred from code, tests, or wiki:
- Product behavior tradeoffs ("should the fee apply to existing positions?")
- Destructive data operations
- Credentials or external access
- Mutually exclusive business requirements
- Accepting a security/data-loss risk

**Do not ask** the user about: which module to edit, whether to update the wiki, whether to include a newly discovered dependency, whether to expand the dev plan. The skill owns those decisions.

### P6. Confidence is explicit
Every module doc, every code-map entry, every interaction edge carries a `confidence` field (`high|medium|low|unknown`). Hidden uncertainty causes cascading damage; recorded uncertainty does not.

### P7. Stable IDs, mutable names
Module IDs (`M0000-root`, `M0001-<slug>`, …) are stable and never renumbered or reused. Titles can change. Splits keep the parent ID and create new child IDs. Merges preserve the old doc with `status: merged` and point to the survivor. This is what keeps dev plans, risks, and interaction maps from rotting.

### P8. Wiki sync is part of "task done", not after it
A dev task is **not complete** until the wiki reflects the new code state. This is a hard gate, not a guideline. Stale wiki is worse than no wiki — it actively misleads the next dev task.

Concretely: every code change that affects **responsibility / state / interactions / public surface / invariants / failure modes / tests / external boundaries / generators** MUST be reflected in the wiki in the same logical unit of work. Not "later". Not "next pass". The same unit.

This applies symmetrically to all change shapes:
- **Modify** an existing module → bump `last_verified`, update affected sections.
- **Add** a new module → write its full `MODULE.md`, update MANIFEST tree + index + concern index, update parent's "Owns/Does Not Own", update interactions / inventories / `progress.json` stats.
- **Remove** a module → set `status: deprecated` (do not delete file), prune MANIFEST tree, scrub cross-references.
- **Rename** a module → keep stable ID, change `title` only.

The exhaustive change-type → sync-actions table and completion gate live in `references/development-protocol.md` Phase 8 (Mandatory Wiki Sync).

### P9. Production-grade, not demo-grade
Every wiki file is read under stress, possibly months later, by future Agents (and the user). Each must be self-contained: a reader landing on `modules/M0042-pow-difficulty/M.md` cold should understand the module without reading anything else. No "see above", no implicit context. No emojis.

---

## Operating discipline (Always / Never)

### Always
1. Prefer `rg --files` and `rg <pattern>` over `find` and `grep` for discovery — faster and gitignore-aware.
2. Preserve the user's existing worktree changes. Never stash, reset, or overwrite uncommitted work.
3. Record evidence for every architectural claim (file:line, symbol, command, config entry).
4. Use stable module IDs. Never renumber existing IDs.
5. Document file **ranges**, not just file paths. One file can contain several modules.
6. Record interaction edges between modules with explicit type (`call|event|data|config|build|network|storage|test|runtime`).
7. Mark `confidence` explicitly on every claim.
8. Refresh line ranges (re-grep anchors) before editing code.
9. Update `.architecture/` **before** expanding the allowed edit surface.
10. Communicate user-facing progress in the user's language (default Chinese for this user).

### Never
- Edit code outside the active dev plan's allowed edit surface.
- Treat generated/vendored files as source of truth unless the repo clearly says they are hand-authored.
- Ask the user to approve technical scope expansion caused by undocumented code. Self-heal and continue.
- Hide uncertainty. Record it in `RISKS.md` or the relevant module doc.
- Collapse a large project into one giant markdown file.
- Rewrite unrelated modules opportunistically for cleanliness.
- Renumber, reuse, or delete a module ID. Use `status: merged|deprecated` instead.

---

## Reference loading (load only what's needed)

| Situation | Files to load |
|---|---|
| Full mapping run | `references/analysis-protocol.md`, `references/boundary-and-evidence-rules.md` |
| Code change request | `references/development-protocol.md`, `references/boundary-and-evidence-rules.md` |
| Divergence detected during work | `references/self-healing-protocol.md` (in addition to the above) |
| Writing a wiki file | the matching `templates/*.md` |

---

## Mode quickflows

### MAPPING mode

1. Create `.architecture/` if missing.
2. Read `references/analysis-protocol.md`.
3. **Phase 0/1** — preflight + census: write `.architecture/.meta/preflight.json` (repo metadata) + initialize `RISKS.md` with any anomalies surfaced (stale docs, placeholder build steps, vendored license boundaries, etc.) + plan inventory categories.
4. **Phase 2a** — top-level module cut. Do not target a count — apply the criteria in `analysis-protocol.md` Phase 2a and let the project's actual seams determine the number. Allocate stable IDs with headroom (`M0001+` top-level, `M0010+` / `M0020+` / … for each top-level's children).
5. **Phase 2b** — full tree planning to leaf level. Assign every stable ID before writing any per-module doc content. MANIFEST `Module Tree` is complete after this phase.
6. **Phase 2.5** — write the 4 inventories under `inventories/`. Inventory rows cite stable IDs from MANIFEST.
7. **Phase 3** — leaf-criteria verification. For each leaf candidate, confirm all leaf criteria hold; if any fails, return to Phase 2b and subdivide.
8. **Phase 4** — fill module docs: **non-leaves first** (establish module-wide invariants), **then leaves** (reference parent invariants by ID). Every `modules/M*.md` ends with the END-OF-MODULE sentinel.
9. **Phase 5** — stitch global interaction maps under `interactions/`.
10. **Phase 6/7** — audit coverage → write `COVERAGE.md`. Audit risks → finalize `RISKS.md` (aggregate from each MODULE.md `## Unknowns And Risks`).
11. **Phase 7 end** — write `.architecture/README.md`; write `.architecture/.meta/progress.json` as a one-shot completion snapshot; finalize `analysis-runs/<...>.md` with the `Finished:` timestamp.
12. Final report to the user: ≤10 lines summarizing modules mapped, wiki location, top risks, and the suggestion that future dev tasks will use this wiki to stay safe.

### DEVELOPMENT mode

1. Read `references/development-protocol.md`.
2. Load `MANIFEST.md`, `COVERAGE.md`, `RISKS.md`, the relevant module docs, and relevant interaction maps. (Do **not** load the whole `.architecture/` tree.)
3. If no candidate modules match the request in `MANIFEST.md`, apply the **Phase 1.5 three-scenario decision rule** (in `development-protocol.md`):
   - **Scenario A** — `rg`/`grep` finds the feature in code but it's unmapped → SELF-HEAL, then resume.
   - **Scenario B** — feature doesn't exist but fits the project's scope → plan as new-module development (autonomous, write the new `MODULE.md` post-implementation).
   - **Scenario C** — feature is outside the project's scope (semantically conflicts with what `MANIFEST.md` says the project is) → pause and ask the user. This is the only autonomous-mode exception.
4. Write a concrete execution plan under `dev-plans/YYYY-MM-DD-HHMM-<slug>.md` per `templates/dev-plan.md`.
5. The plan names: target modules, allowed edit surface, forbidden nearby areas, expected interaction changes, self-healing triggers, verification commands.
6. Execute the plan directly. Do not ask the user for technical confirmation.
7. On divergence → SELF-HEALING mode, then resume.
8. Update module docs and interaction maps if architecture behavior changed.
9. Run targeted verification first; broader verification based on blast radius (see boundary-and-evidence rules).
10. Final report: dev plan path, changed modules, wiki updates, verification results, remaining risks.

### SELF-HEALING mode (auto, inside either mode)

1. Stop expanding the code edit.
2. Read `references/self-healing-protocol.md`.
3. Analyze the discovered area enough to make the current change safe.
4. Update `modules/`, `interactions/`, `COVERAGE.md`, `RISKS.md` as appropriate.
5. Write `self-healing/YYYY-MM-DD-HHMM-<trigger-slug>.md` per `templates/self-healing-report.md`.
6. Update the active dev plan (new allowed surface, new forbidden areas, new verification).
7. Continue. Inform the user in 1–2 sentences.

---

## Completion criteria

**Mapping is complete** when:
- Every first-party code area is `covered`, `partially covered` with reason, or explicitly `excluded`.
- Every leaf module has a range-level code map and interaction summary.
- Risks are recorded in `RISKS.md`, not hidden.
- Every `modules/M*.md` ends with the END-OF-MODULE sentinel (see `templates/module.md`) — this distinguishes complete files from files interrupted mid-write.
- `progress.json` is written ONCE at this point as a completion snapshot (`"complete": true` + stats + `next_frontier`). It is not maintained during execution.

**Development is complete** when:
- The dev plan was written before any code edit.
- All edits stayed inside documented or self-healed ranges.
- Wiki reflects the new state of changed modules.
- Targeted and (when needed) broader verification ran successfully, or failures are recorded as risks.
- Final report names what changed, what was healed, and what remains risky.

---

## What this skill is NOT

- Not a quick scan. Mapping is intentionally slow and exhaustive on a large repo. Expect hours.
- Not a code-review tool. For that, use `/review`.
- Not a planner for greenfield projects. Use `office-hours` + `plan-eng-review`.
- Not a substitute for tests. The wiki says *what* to change; tests say whether you *broke* something.
- Not a security auditor. Funds/protocol/consensus changes still need real audit.
