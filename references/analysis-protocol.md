# Architecture Analysis Protocol

Use this protocol to turn an unfamiliar repository into a durable `.architecture/` wiki that lets a future Agent modify a small module without accidentally changing unrelated behavior.

**The output is not a narrative overview.** It is an operational control plane for safe development. Every claim carries evidence. Every uncertainty is recorded.

This phase is intentionally slow on a medium-large project (50k–500k LOC). Expect many tool calls and long sessions. Do not rush. Do not skip. The user has explicitly traded time for the guarantee that no Agent will silently break a contract in a module the Agent didn't know existed.

---

## Phase 0 — Establish repository state

Before any analysis:

1. **Identify repository root**: `git rev-parse --show-toplevel`, fall back to `pwd` if not a git repo.
2. **Worktree status**: `git status --short`. Note uncommitted changes — never overwrite them.
3. **Current branch + commit**: `git branch --show-current` and `git rev-parse HEAD`.
4. **Repo origin** (if any): `git remote -v` — useful for citing external docs.
5. **Detect**: languages, package managers, build files, test files, config files, generated outputs, vendored deps, docs.

Write `.architecture/.meta/preflight.json` with the structured snapshot:

```json
{
  "repo_root": "<abs path>",
  "branch": "<name>",
  "commit": "<sha>",
  "analyzed_at": "<ISO8601>",
  "languages": ["go", "solidity", ...],
  "package_managers": ["go.mod", "foundry.toml", ...],
  "build_systems": ["make", "foundry", ...],
  "entry_points_candidates": ["cmd/node/main.go", ...],
  "test_dirs": ["tests/", "contracts/test/", ...],
  "generated_paths": ["pb/", "abi/", ...],
  "vendored_paths": ["vendor/", "third_party/", ...],
  "doc_paths": ["docs/", "README.md", ...],
  "loc_total": 0,
  "loc_by_top_dir": {"cmd": 0, "internal": 0, ...},
  "file_count": 0
}
```

Then record this run's start in `analysis-runs/YYYY-MM-DD-HHMM-initial.md` (use `templates/analysis-run.md`).

---

## Phase 1 — Repository census

This is the `/init`-style fast onboarding pass — get the *shape* of the repo before subdividing.

### 1.1 Read the project's own onboarding artifacts

These tell you what the maintainers think matters:
- `README.md`, `CONTRIBUTING.md`, `ARCHITECTURE.md`, `DESIGN.md`, `docs/`, `RFCs/`
- `Makefile`, `justfile`, `package.json` scripts, `pyproject.toml` tool sections, `Cargo.toml`, `go.mod`, `foundry.toml`, `hardhat.config.*`, `truffle-config.*`
- `CHANGELOG.md` — recent activity hints at hot areas
- CI configs (`.github/workflows/`, `.circleci/`, `.gitlab-ci.yml`) — they reveal real test/build commands

### 1.2 Map the tree shape

```bash
rg --files 2>/dev/null | head -3000   # gitignore-aware
# fallback: find . -type f -not -path './node_modules/*' -not -path './.git/*' | head -3000

rg --files | awk -F/ '{print $1}' | sort | uniq -c | sort -rn  # file count per top dir
```

For LOC, prefer `tokei` or `scc` if available; else `wc -l` on the file list.

### 1.3 Plan inventory categories (do not write files yet)

Decide which of the 4 standard inventories the project needs:

- **`entrypoints`** — `main`, `index`, `app`, server bootstrap, CLI definitions, contract `constructor`s, scheduled job registration.
- **`tests`** — test framework(s), test directories, test commands from CI/Makefile.
- **`external-interfaces`** — HTTP routes, gRPC services, RPC methods, CLI subcommands, contract ABI methods, event signatures, exported library symbols.
- **`generated-and-vendor`** — generated code paths + generator command/source; vendored dirs + the upstream they vendor.

All 4 are recommended for any non-trivial codebase. **The files themselves are not yet written** — they are written in Phase 2.5, after the MANIFEST tree assigns stable module IDs. Inventory rows must cite module IDs (`M####-<slug>`) in their `Related module(s)` column, and forcing those IDs to exist before the inventory is written is what makes Phase 2.5 the right home.

### 1.4 Initialize `RISKS.md` early (and append throughout)

`RISKS.md` is **the canonical event log of project-level risk** discovered during analysis. Initialize it now from Phase 0/1 preflight findings — every later phase appends to it.

Initialize with rows for any of these surfaced during Phase 0/1 census:

- Source-tree anomalies (sync conflict files, orphaned files, files with mismatched extensions).
- Stale top-level documentation (README / CLAUDE.md / AGENTS.md / project guide drift vs actual tree).
- Placeholder build steps (e.g., a `lint` script that just echoes; a `test` script that always passes).
- Generated artifacts whose generator is not yet identified.
- License-isolated subtrees (vendored or upstream snapshots with explicit license boundary).
- Known-fragile external systems the project depends on without a fallback.
- Active investigation / refactor threads recorded in project memory or RFC docs (these signal upcoming churn).

After Phase 1, `RISKS.md` is the single source of truth for risk. Per-module docs may reference these risks by ID (`R####`); the master copy lives here.

### 1.5 Evidence required for everything

A directory name alone is **not** evidence. Acceptable evidence:
- Import graphs (resolved via static parse or `rg` on import statements)
- Route/command/service registration code (the actual line that calls `app.register(...)`)
- Package manifests
- Build configuration
- Protocol / ABI declarations
- Database migrations
- Test fixtures
- Runtime wiring (the place where dependencies are constructed and passed)
- Public exports (`__all__`, `mod.rs` re-exports, `index.ts` re-exports, package public API)
- README commands that match actual files

If a claim has no evidence, mark it `confidence: low` or `unknown` and either chase down evidence or accept it as a risk.

---

## Phase 2a — Top-level module cut

Create the first-pass top-level module hypothesis in `.architecture/MANIFEST.md` using `templates/manifest.md`. **This phase produces only the top-level row of the Module Tree.** Recursive subdivision into children is Phase 2b.

**Do not target a specific module count.** Let the project itself determine the cut. A valid top-level module has a distinct reason to exist — apply these criteria honestly and the count falls where it falls:

- Owns a protocol layer (consensus, p2p, mempool, RPC).
- Owns persistent state (storage engine, chain state, account DB).
- Owns an API boundary (HTTP/gRPC/JSON-RPC server, public SDK surface).
- Owns execution/runtime orchestration (node bootstrap, scheduler, worker pool).
- Owns build/deployment tooling (codegen, contract compilation, deployment scripts).
- Owns user-facing workflow (wallet, CLI commands, dashboard).
- Owns integration with an external system (oracle adapter, bridge, indexer feed).
- Owns tests or fixtures for a major subsystem.

### Sanity rails (catch egregious failures, not anchor a target)

After your first cut, audit against these two boundaries. They are **failure detectors**, not targets — do not adjust the cut just to fall inside the band.

- **Fewer than 3 top-level modules?** You have not decomposed yet. Re-examine — almost no real project's responsibilities collapse this far. Look for hidden seams (build vs runtime, persistent state vs in-memory state, public API vs internal core).
- **More than ~25 top-level modules?** You are likely conflating depth with breadth — several of these are probably children of broader concerns. Try one round of grouping: which of these share a lifecycle, share state, or change for the same reasons?

Both extremes are signals to re-examine, not to mechanically merge or split. A tiny utility legitimately has 3 modules. A kubernetes-scale monorepo legitimately has 30+. The criteria above govern; these rails only flag when you may have applied them lazily.

**Common domain patterns** (use as hints, not as a checklist):

- **Blockchain node**: consensus, p2p, mempool, state/storage, RPC/API, crypto primitives, VM / contract execution, sync, wallet/keys, observability, governance.
- **Smart contract project**: token logic, access control, governance, treasury, oracle integration, upgrade proxy, ABI bindings, deploy scripts, fork tests.
- **Web app / SaaS**: routing, auth, persistence, business domain modules, integrations, UI, jobs, observability.
- **Library/SDK**: public API surface, internal core, transport, codec/serialization, retries, types/models.

Each top-level module starts as `status: hypothesis` with `confidence: low` until code evidence is gathered. Assign stable IDs at this point (`M0001-<slug>`, `M0002-<slug>`, …). `M0000-root` is the conceptual root.

**Important**: when allocating IDs, use **densely-numbered top-level IDs** (`M0001`, `M0002`, …) but **leave headroom for children**: each top-level module's children will start at the next round-number base (`M0010+` for first top-level's children, `M0020+` for second's, etc. — or `M0020+` / `M0030+` if the first top-level needs many children). Gaps between groups let later self-healing add child modules without losing the namespace grouping. A future `M0083` should obviously belong to a known top-level group; `M0048` should obviously belong to a different one. That semantic mapping survives even when the MANIFEST is read out of order.

---

## Phase 2b — Full tree planning (assign all stable IDs before writing leaf content)

Now extend the MANIFEST `Module Tree` and `Module Index` recursively to the **leaf level**, but **do not yet write the per-module `MODULE.md` content**. Goal: pin down every stable ID and the parent/child structure in one pass, so that:

- Per-module docs (Phase 4) can reference children/siblings by stable ID from the start.
- Inventories (Phase 2.5) can cite stable IDs.
- A session interrupted between 2b and 4 has a complete planning artifact to resume from.

For each top-level module from 2a, recurse:

1. **Re-anchor on code**: enumerate the files and symbols belonging to this module. Use `rg -n` to map cross-file references. Read entry-point files of the node fully — enough to identify sub-responsibility seams.
2. **Identify seams**: where does this node have natural sub-responsibilities? (Separate sub-packages, separate top-level classes, distinct lifecycle stages, different external collaborators, independent test files.)
3. **Apply split triggers** (see Phase 3 — recursive decomposition for the full list).
4. **Subdivide**: assign next stable IDs (incrementing through that top-level module's headroom block), record parent/child relationships in `MANIFEST.md`. Each new child starts as `status: hypothesis, confidence: low`.
5. **Or finalize as leaf**: apply leaf criteria (see Phase 3 + self-healing-protocol leaf-criteria triggers).
6. Continue recursion until every branch reaches a leaf candidate.

By the end of Phase 2b, `MANIFEST.md`'s `Module Tree` is **complete to the leaf level** — every node has a stable ID; no leaf body content is written yet.

---

## Phase 2.5 — Build inventories (after MANIFEST tree, before leaf docs)

With every stable ID now allocated in MANIFEST, write the inventory files planned in Phase 1.3.

Create one file per category under `inventories/` using `templates/inventory.md`:

- `inventories/entrypoints.md`
- `inventories/tests.md`
- `inventories/external-interfaces.md`
- `inventories/generated-and-vendor.md`

Each row's `Related module(s)` column **must** cite the stable ID(s) from `MANIFEST.md`. Use **anchors** (symbol / route / script name / config key) rather than line ranges that will drift. Inventory rows that reference an external system not yet documented should also be added to `interactions/external-boundaries.md` in Phase 5.

This phase runs after MANIFEST tree because inventory items need to cross-reference module IDs that don't exist until the tree is decided. Running it earlier produces inventories full of placeholder IDs that must be back-filled — error-prone and wasteful.

---

## Phase 3 — Leaf-criteria verification (final tree check before writing docs)

By the end of Phase 2b every branch of the tree reached a leaf candidate. Phase 3 is the **final verification pass** before per-module docs are written: for each leaf candidate, confirm that all leaf criteria below hold. If any fails, return to Phase 2b for further subdivision (assign new IDs, update MANIFEST tree).

This phase exists as a guard against "ran out of patience and called it a leaf" — the failure mode this entire skill exists to prevent. The leaf criteria are also the canonical reference for the self-healing trigger "leaf-criteria self-check fail" (see `self-healing-protocol.md`).

**There is no depth limit.** Some subtrees stop at depth 1, others at depth 4 or more. Honest application of the criteria below determines the depth.

### Split triggers — split further when ANY is true

- Owns multiple independent runtime responsibilities.
- Different files or ranges change for different reasons.
- Has separate state ownership areas.
- Has separate inbound interfaces (different external callers).
- Has separate outbound dependencies.
- Mixes policy, parsing, validation, persistence, networking, scheduling, or UI concerns.
- A plausible future change request would touch only part of it.
- Its explanation requires multiple unrelated mental models.
- Its test coverage naturally splits into separate behaviors.
- Its code map would contain broad ranges (>500 lines per range) that are unsafe as edit boundaries.

### Leaf criteria — mark as leaf only when ALL are true

- Its responsibility can be explained precisely in one or two sentences.
- Its implementation locations are known at file + line-range level.
- Its line ranges have **stable anchors** (symbol/route/config-key/test name/ABI name) — never just line numbers.
- Its upstream callers (inbound) are documented with evidence.
- Its downstream dependencies (outbound) are documented with evidence.
- Its state ownership and side effects are documented.
- Its important invariants and failure modes are documented.
- Its tests or verification commands are documented (or the absence is recorded as a risk).
- Nearby non-owned code is identified ("does not own" section in `templates/module.md`).
- A future Agent can identify, from this doc alone, where to edit and where not to edit.

**Do not mark a module as leaf merely because time is running out.** That is the failure mode this skill exists to prevent.

### Progress tracking & resume — filesystem-based, not journal-based

`.architecture/.meta/progress.json` is a **completion snapshot** written **once** at the very end of Phase 7. It is NOT a live execution journal. Do not attempt to maintain it incrementally — that is unreliable across long runs.

Resume after interruption uses **filesystem state**, not the JSON.

#### Resume protocol (when invoked with existing `MANIFEST.md` but missing terminal artifacts)

1. **Read `MANIFEST.md`** — this is the canonical planned tree. Every node in `Module Index` is a target.
2. **Scan `modules/`** — list existing `M*.md` files.
3. **Validate each existing file**: check it ends with the END-OF-MODULE sentinel (see `templates/module.md` — a literal trailing comment line). A file without the sentinel is **interrupted-mid-write** and must be **rewritten from scratch** (idempotent re-write).
4. **Compute the gap** = `MANIFEST.md` `Module Index` IDs minus the set of validated `modules/M*.md` files. This is the work queue.
5. **Process the gap** in MANIFEST order. Top-down (non-leaves first), then leaves, per Phase 4 ordering.
6. **Also check terminal artifacts**: if `interactions/`, `inventories/`, `COVERAGE.md`, `RISKS.md`, `README.md` are missing, queue Phases 5–7 after leaves complete.

#### `progress.json` schema (written once, at completion)

```json
{
  "complete": true,
  "started_at": "<ISO8601>",
  "updated_at": "<ISO8601 of final write>",
  "next_id": 80,
  "stats": {
    "modules_total": 64,
    "leaves_done": 57,
    "non_leaves": 7,
    "inventories": 4,
    "interactions": 4,
    "decisions": 0,
    "analysis_runs": 1,
    "risks_recorded": 30
  },
  "next_frontier": [
    "Deep-dive area X if/when it stabilizes",
    "Refresh affected leaves after refactor Y lands"
  ]
}
```

Notes:
- `next_id` reflects the **next free** stable ID to assign. New modules added by future development MUST increment from here.
- `stats` is informational. Counts here should match what is on disk; they are not authoritative if a discrepancy is found (the disk is authoritative).
- `next_frontier` is forward-looking. Each entry describes a known incomplete area or a deferred deep-dive — recorded so the next analysis run knows where to pick up.

If a session aborts before `progress.json` is written, the next run reads MANIFEST + filesystem and resumes — the absent `progress.json` is itself a signal that mapping is incomplete.

---

## Phase 4 — Module documentation

Every leaf module's `MODULE.md` (under `modules/M{id}-{slug}.md`) MUST include the full set of sections from `templates/module.md`:

- frontmatter: `id`, `title`, `parent`, `status`, `leaf`, `confidence`, `last_verified`
- Summary
- Architecture Role
- Scope: **Owns** and **Does Not Own**
- Code Map (table with role / file / lines / anchor / edit guidance / confidence)
- Inbound Interactions (with edge type)
- Outbound Interactions (with edge type)
- State And Data Ownership
- Invariants
- Failure Modes
- Tests And Verification
- Modification Guide (safe / requires self-healing / avoid)
- Leaf Certificate (checklist — only for leaves)
- Unknowns And Risks
- Evidence Log

If a section genuinely doesn't apply (e.g. a stateless pure-function module), write `_(none)_` — never omit. Omission is indistinguishable from oversight.

For non-leaf modules, fields like Code Map can be sparse (point at child modules instead), but Summary, Architecture Role, and the Owns/Does Not Own scope MUST be filled.

### Phase 4 ordering: non-leaf first, then leaf

Write per-module docs in two batches:

1. **Non-leaf docs first** (top-down). Each non-leaf doc establishes module-wide invariants, the Owns / Does Not Own scope, the child subtree map, module-wide failure modes, and a Modification Guide that applies to all descendants.
2. **Leaf docs second**. Each leaf doc references its parent's invariants by stable ID (e.g., "subject to M0001-hub's no-direct-Repository-imports rule") rather than restating them. This keeps the wiki DRY and consistent across siblings.

If self-healing later changes a parent invariant, only the parent doc needs updating — descendant leaf docs that referenced the parent automatically inherit the new invariant via the cross-reference. Restating invariants in every leaf would force a cascade of edits.

Within each batch, parallel subagents per top-level subtree are appropriate (see "Subagent strategy" below). Files within a batch may be written in parallel; the subagent rule (only main agent writes `.architecture/`) still applies.

### Phase 4 sentinel requirement

Every `modules/M*.md` MUST end with the END-OF-MODULE sentinel (the trailing HTML comment in `templates/module.md`). A file lacking the sentinel is treated as interrupted-mid-write by the resume protocol — it will be rewritten from scratch on the next session. Do not skip the sentinel because a file "looks complete enough".

---

## Phase 5 — Interaction stitching

Create global interaction maps under `interactions/`, one file per category, using `templates/interaction-map.md`:

- **`runtime.md`** — runtime calls and ownership (who instantiates whom; lifecycle).
- **`data-flow.md`** — data ownership and transformations (where data is born, transformed, persisted).
- **`build-and-config.md`** — build-time generation, config-controlled wiring.
- **`external-boundaries.md`** — network/storage/external-system edges.

**Edge types** (use consistently across all docs):
- `call` — direct function/method/module call.
- `event` — event bus, callback, hook, signal, observer, pub/sub.
- `data` — reads or writes shared data structure or store.
- `config` — behavior controlled by config value.
- `build` — build-time or generation dependency.
- `network` — HTTP, RPC, p2p, websocket, queue, external protocol.
- `storage` — DB, FS, object storage, chain state, cache.
- `test` — tests or fixtures defining expected behavior.
- `runtime` — process lifecycle, scheduler, worker, daemon, plugin loader.

The same edge may appear in a module doc and a global map. Duplication is intentional: module docs support local edits; global maps support blast-radius reasoning.

---

## Phase 6 — Coverage audit

Update `.architecture/COVERAGE.md` per `templates/coverage.md`.

Classify every first-party file as:
- **covered** — owned by a documented leaf module.
- **partially covered** — some ranges owned, others not. Document what's covered and what isn't.
- **excluded** — generated / vendored / build artifact / binary / third-party copy. Reason required.
- **unknown** — file exists but no module claims it. **This is a frontier**; either claim it in the next pass or record as a risk.

A wiki where some code is `unknown` is acceptable. A wiki that pretends all code is `covered` when it isn't is worse than useless.

---

## Phase 7 — Risk audit (aggregate from authoritative source only)

`RISKS.md` was initialized in Phase 1.4 with preflight anomalies. Phase 7 is the **aggregation pass** that pulls per-module risks discovered during Phases 4–6 into the master file.

### Aggregation rule

The **only authoritative per-module risk source** is each module's `## Unknowns And Risks` section. The Phase 7 aggregator MUST scan every `modules/M*.md` and lift every row of `## Unknowns And Risks` into `RISKS.md` Active Risks (assigning a new `R####` ID per item).

Code Map "Notes" columns, Inbound/Outbound "Notes", and other free-text annotations are **NOT** scanned by the aggregator. If a writer noted a warning in a Code Map row (e.g., "verify-later", "drop-named", "potential drift"), they MUST also have promoted it to `## Unknowns And Risks` in the same module doc when they wrote it (per the template Code Map note). Phase 7 does not search the whole module doc for warning prose — that would be unreliable and produce false positives.

### Per-module aggregation procedure

For every `modules/M*.md`, in MANIFEST order:

1. Read the `## Unknowns And Risks` section.
2. For each bullet, draft a `RISKS.md` Active Risks row:
   - `R####` — next free risk ID (the master RISKS.md tracks this).
   - `Area` — the source module's stable ID (`M####-<slug>`).
   - `Risk` — the concrete concern, restated in one sentence.
   - `Severity` — `low` | `medium` | `high` | `critical`. Default to `medium` if unsure; downgrade only with explicit reason.
   - `Confidence` — how sure the aggregator is this is actually a risk.
   - `Mitigation` — `investigate` / `monitor` / `accepted` / a concrete plan / `defer to phase-2`.
   - `Owner` — the source module ID (same as Area unless ownership is shared).
3. If the bullet duplicates an existing risk in `RISKS.md` (e.g., the same gap mentioned in two modules), merge — extend the existing row's `Area` column to list both module IDs.
4. After aggregation, walk the new `RISKS.md` Active Risks. Surface to the user in the final report any `severity: high|critical` items.

### Additional Phase 7 sources

Beyond `## Unknowns And Risks` aggregation, also audit and add rows for:

- Low-confidence modules (`confidence: low` or `unknown` in frontmatter).
- Inferred behavior not verified by tests.
- Dynamic dispatch / reflection / plugin loaders that hide static structure.
- Generated code whose generator source is unclear or missing.
- External systems not available for verification (testnets, oracles, third-party APIs).
- Circular dependencies.
- Broad edit ranges (single anchor spans > ~500 LOC).
- Missing tests on critical modules.
- Stale docs (line ranges already out of sync at the time of writing).
- Modules with no inbound and no outbound edges that are not entrypoints (suspect dead code).

Risks are not failures. Hidden risks are failures.

---

## Architectural decisions

When the analysis makes a judgment call that is not obvious — splitting a module a non-obvious way, naming choice, classifying something as excluded, treating something as a leaf despite size — write a `decisions/YYYY-MM-DD-HHMM-<slug>.md` entry using `templates/decision-record.md`. This preserves the reasoning so future analysis (and dev work) can re-evaluate if context changes. Skip this for obvious choices; reserve it for forks in the road.

---

## Subagent strategy

For non-trivial repos, **dispatch each top-level (round-2) module to a dedicated Agent subagent in parallel**. Rules:

- Give each subagent: SKILL.md content (or at minimum the principles) + this protocol + `references/boundary-and-evidence-rules.md` + `templates/module.md` + `preflight.json` + its assigned subtree path + its assigned ID range (e.g. `M0010-M0019`).
- Subagents must **not** edit source code.
- Subagents must **not** write to `.architecture/` directly. They return: proposed module structure (IDs + slugs + hypothesized children), evidence collected, code-map drafts, risks/unknowns found.
- The orchestrator (you) owns final `.architecture/` writes — this prevents parallel writes from clobbering shared files like `MANIFEST.md`, `COVERAGE.md`, `RISKS.md`.
- Do not let parallel subagents work on overlapping subtrees.

For smaller repos, do everything in one session.

### Deferred items pattern

When the orchestrator needs to note an item that requires deeper subagent analysis but cannot wait for that analysis to finish — e.g., inventory rows whose evidence will come from the subagent's deep-dive — use this pattern:

1. Mark the item in-place with `(TBD: deep-dive on M####-<slug>)` — include the module ID the subagent is responsible for.
2. Add an entry to `.architecture/.meta/progress.json` under `deferred_items` (one row per TBD):
   ```json
   "deferred_items": [
     {
       "file": "inventories/entrypoints.md",
       "row_anchor": "runtime/<package-name>",
       "pending_on": "M####-<slug>",
       "added_at": "<ISO8601>"
     }
   ]
   ```
3. When the responsible subagent finishes its module doc, the orchestrator backfills the deferred item using the subagent's report and removes the row from `deferred_items`.
4. Mapping is NOT complete until `deferred_items` is empty. The Stopping condition checks this.

Without this pattern, "(TBD)" notes in published wiki files become silent technical debt — no one circles back to resolve them.

---

## Inter-module reference integrity (final pass)

After all leaf modules are written:

1. For every `outbound` entry across all modules, verify the target module exists in `MANIFEST.md`.
2. For every target, verify the reciprocal entry exists in the target's `inbound`.
3. Any asymmetry = a bug. Fix it.
4. Modules with neither inbound nor outbound and not an entrypoint → suspect dead code. Flag in `RISKS.md` rather than deleting analysis.

---

## Stopping condition

Write `progress.json` (one shot, completion snapshot) only after **all** of:

- `.architecture/README.md` written (from `templates/architecture-readme.md`).
- `.architecture/MANIFEST.md` written and lists every module ID with status.
- `.architecture/COVERAGE.md` classifies every first-party file.
- `.architecture/RISKS.md` records all known uncertainty.
- `modules/M*.md` written for every module in the tree, **and every file ends with the END-OF-MODULE sentinel** (`<!-- END OF MODULE DOC -->`). Files missing the sentinel were interrupted mid-write — re-write them before completion.
- `interactions/*.md` written for the 4 standard maps.
- `inventories/*.md` written for entrypoints, tests, external interfaces, generated/vendor.
- Analysis run record written in `analysis-runs/` with `Finished:` timestamp filled in.
- All inbound/outbound references resolve symmetrically across modules.

If the repo is too large to finish in one session, leave a coherent partial wiki: every existing `modules/M*.md` must be sentinel-terminated (no half-finished files), and `COVERAGE.md` / `RISKS.md` are not yet required. The absence of `progress.json` is itself the signal that mapping is incomplete — the next session resumes by filesystem scan.

---

## Final report to the user

≤ 10 lines:
- Total modules mapped (leaves / non-leaves).
- Wiki location: `.architecture/`.
- Coverage summary (covered / partial / unknown / excluded counts).
- Top risks flagged.
- Hint: "Next time you ask me to modify or extend this project, I'll use this wiki to keep changes safe."

Then stop. Do not propose changes. Do not start development mode unprompted.
