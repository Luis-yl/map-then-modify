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
  "repo_name": "<name>",
  "branch": "<name>",
  "commit": "<sha>",
  "analyzed_at": "<ISO8601>",
  "languages": ["..."],
  "package_managers": ["..."],
  "build_systems": ["..."],
  "entry_points_candidates": ["..."],
  "test_dirs": ["..."],
  "generated_paths": ["..."],
  "vendored_paths": ["..."],
  "doc_paths": ["..."],
  "loc_total": 0,
  "loc_by_top_dir": {"...": 0},
  "file_count": 0,
  "preflight_warnings": [
    "<each entry: a project-self-issue surfaced during the census — for example: stale top-level documentation drift vs current tree, placeholder build/lint scripts that look real but do nothing, sync-conflict files in source tree, license-isolated subtrees, etc.>"
  ]
}
```

**About `preflight_warnings`**: this field is **mandatory** even if empty. Real projects almost always have one — every `preflight_warnings` row also becomes an initial entry in `RISKS.md` (Phase 1.4). Capturing these at the very start means later phases work from a known-stale baseline, not pretending the project is pristine.

Optional fields when relevant to the project's domain — include them when they would matter for future dev tasks:

- `timezone_pinned` — a project-level invariant on time handling (e.g., `"Asia/Shanghai (set at src/main.ts:2)"`). Time-sensitive projects break in subtle ways without this.
- `non_workspace_runtime_dirs` — in a monorepo, top-level directories that contain meaningful code but are not declared workspaces. They are easy to miss in the cut.
- `active_research_threads` — pointers to in-flight refactor / investigation work (research docs, RFCs, project memory) that will likely churn parts of the architecture in coming weeks. Signals upcoming wiki staleness.

If a field genuinely doesn't apply, omit it — these are project-shape dependent. The mandatory fields above are not optional.

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

**Common domain patterns** (use as hints, not as a checklist; deeper per-domain guidance lives in `references/domain-heuristics/`):

- **Blockchain node**: consensus, p2p, mempool, state/storage, RPC/API, crypto primitives, VM / contract execution, sync, wallet/keys, observability, governance. → See `references/domain-heuristics/blockchain-node.md` for typical modules, boundary pitfalls, and risk hotspots.
- **Smart contract project**: token logic, access control, governance, treasury, oracle integration, upgrade proxy, ABI bindings, deploy scripts, fork tests. → See `references/domain-heuristics/smart-contract.md`.
- **Web app / SaaS**: routing, auth, persistence, business domain modules, integrations, UI, jobs, observability. → See `references/domain-heuristics/web-saas.md`.
- **Library/SDK**: public API surface, internal core, transport, codec/serialization, retries, types/models. → See `references/domain-heuristics/library-sdk.md`.
- **CLI tool**: command parser, commands, config, runtime context, output, state, integrations. → See `references/domain-heuristics/cli-tool.md`.

### Domain heuristic loading

After Phase 0/1 preflight completes, check `preflight.json` against the loading rules in `references/domain-heuristics/README.md` ("When to load") and read the matching file(s) for any domain(s) that apply.

Heuristic files are **references for comparison**, not templates to copy. For every typical top-level module listed in the heuristic, ask "does this project have this responsibility, and have I captured it?". For every top-level module YOU propose that the heuristic does NOT list, justify it briefly in the top-level decision record. This forces non-standard cuts to be deliberate, not accidental.

A project matching NO heuristic file (research code, ML pipeline, mobile app, embedded firmware, game engine, etc.) — do not invent a heuristic. Use the Phase 2a criteria directly. Note the missing heuristic in `progress.json.next_frontier` as a candidate for the library to grow.

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

Every leaf module's `MODULE.md` (under `modules/M{id}-{slug}.md`) MUST include the full set of sections from `templates/module.md`. For modules that own pure documents (contracts, manuals, policies) rather than executable code, see `templates/examples/module-non-code-leaf.md` for a worked variant using structural anchors instead of line ranges.

Required sections:

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

### Phase 4 leaf-doc strategy: batched writer

For an N-leaf project (often 30–80 leaves), the protocol uses a **batched writer** model rather than one-subagent-per-leaf. The rationale comes from real-world tradeoffs: dispatching 80 separate subagents has high orchestrator-side coordination overhead; coordinating each subagent to know its narrow scope, parent invariants, and cross-references is expensive; and the cost amplifier (token budget per session, wall time, dispatch limits) blows up.

The batched writer model:

1. **Group cohesive leaves into batches of 5–10**. Cohesion criteria (any one suffices):
   - Same parent subtree (e.g., all hub-infra leaves: M0010–M0019)
   - Same subsystem with shared invariants (e.g., the 7 asset family leaves M0030–M0036)
   - Same domain workflow stage (e.g., all knowledge-ingest pipeline stages)
2. **Dispatch one writer subagent per batch**. The writer is given: SKILL.md content, this protocol, `boundary-and-evidence-rules.md`, `templates/module.md`, `preflight.json`, the batch's leaf IDs + their planned scope from MANIFEST, the parent module's doc(s) for context, full source-code access, and the four inventories.
3. **The writer produces all leaf docs in the batch** in one call, writing each to `modules/M####-<slug>.md` with the END-OF-MODULE sentinel.
4. **Multiple batches run in parallel** per the concurrency budget in "Subagent strategy" below.

This is the canonical strategy. It is not "speed mode" or a fallback — it is what the protocol expects. The earlier writer+reviewer-pair model (which appeared in prior protocol drafts) imposed coordination costs that outweighed its quality value in practice; quality is now achieved via Phase 4.4 Quality Gate + Phase 6.2 reverse coverage + Phase 7 risk aggregation working together.

#### When to use a stricter pass (high-risk leaves)

Some leaves merit extra rigor — specifically those in: `auth`, `payment`, `consensus`, `crypto`, `persistence-migration`, `public-API`, `signing`, `governance`. For these, after the batched writer completes their initial doc, the orchestrator does a **read-back verification pass** itself:

1. Re-read the source files cited in the leaf's `code_map`.
2. Spot-check each `Inbound`/`Outbound` row by `rg` against the named symbol.
3. Verify each Invariant has an Evidence Log reference.
4. If any check fails, dispatch a follow-up writer-patch directive (per Quality Gate remediation, below).

This stricter pass is **judgment-based**, not protocol-mandated for every leaf. The orchestrator decides per leaf based on its risk classification.

### Phase 4.4 — Quality Gate (per-leaf scorecard, 6 essential checks)

After a leaf doc is written, before it is considered final, run a deterministic Quality Gate. The Gate guards against "filled-out but hollow" docs.

#### Scoring rules (6 hard checks)

For a leaf doc, every check must pass:

| Check | Pass rule |
|---|---|
| 1. Frontmatter completeness | `id`, `title`, `parent`, `status`, `leaf`, `confidence`, `last_verified`, `last_verified_sha` all present and non-empty |
| 2. Code Map anchors | every `code_map` row has a non-empty `anchor` (symbol / route / section / config-key / test name) — never blank, never `{{placeholder}}` |
| 3. Interactions ≥1+1 | `Inbound` has ≥1 row OR explicit "no inbound — this is an entrypoint" note; AND `Outbound` has ≥1 row OR explicit "no outbound — leaf utility" note |
| 4. Modification Guide three-bucket | all three buckets (Safe / Requires self-healing first / Avoid) have ≥1 concrete entry; "no items" allowed only with one-line reason |
| 5. Leaf Certificate ticked | all 10 checkboxes ticked (only for `leaf: true`) |
| 6. END-OF-MODULE sentinel | trailing sentinel comment present |

Non-leaf modules apply checks 1, 2 (relaxed: 2-column variant per template), 3, 4, and 6. The Leaf Certificate check is leaf-only.

#### Remediation procedure

1. Orchestrator runs the Gate (mechanical — no LLM judgment for pass/fail).
2. For each failing check, emit a specific actionable directive (e.g., "Code Map row 3 has `anchor: {{symbol_or_anchor}}` — replace with the real symbol name found in `path/to/file.ext:N`").
3. Send directives back to the writer for a **targeted patch**, not a full rewrite.
4. Re-run the Gate. If any check still fails after 3 rounds, escalate: mark the leaf `confidence: low`, add an `R####` to `RISKS.md` describing the persistent quality failure, and finalize anyway. A persistent quality failure is itself a project risk worth surfacing.

#### What Quality Gate does NOT check

- Whether Summary is "good prose" — writer's craft, not gate-able mechanically.
- Whether Invariants are "true" — only that they are *present*. Truth verification is handled by Phase 6.2 reverse coverage + Phase 7 risk aggregation cross-referencing.
- Whether Modification Guide gives "good advice" — only that it is *filled* with concrete entries.

The Gate's purpose is to refuse hollow docs, not to second-guess judgment. Combined with reverse coverage (Phase 6.2) and risk aggregation (Phase 7), the three together cover: doc hollowness (4.4), orphaned source files (6.2), and module-level risks that need project-level visibility (7).

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

## Phase 6 — Coverage audit (forward classification + reverse validation)

Phase 6 is a **two-pass coverage check**. Forward classification answers "given a module, what does it own?" — Reverse validation answers the harder question "given a file, who owns it?". The reverse pass is what catches "AI forgot to map an entire area" failures that the forward pass cannot see.

### 6.1 Forward classification (build-up from modules)

Update `.architecture/COVERAGE.md` per `templates/coverage.md`.

Walk every `modules/M*.md`. For each row in its `code_map`, mark the cited file as covered (with the cited line range tracked). After all module docs are walked, classify every first-party file as:

- **covered** — owned by a documented leaf module, full file or with documented ranges.
- **partially covered** — some ranges owned, others not. Document what's covered and what isn't.
- **excluded** — generated / vendored / build artifact / binary / third-party copy. Reason required.
- **unknown** — file exists but no module claims any range of it. **This is a frontier.**

A wiki where some code is `unknown` is acceptable. A wiki that pretends all code is `covered` when it isn't is worse than useless.

### 6.2 Reverse validation (top-down from filesystem)

After forward classification, run reverse validation. Without this, a forgotten subdirectory or hidden subsystem stays invisible — the forward pass only finds what modules cite, never what they don't.

Procedure:

1. **Enumerate first-party files**: `rg --files` (gitignore-aware) → list, minus excluded paths (`generated_paths` + `vendored_paths` + `node_modules` + build artifacts from `preflight.json`).
2. **For each file, query "which module owns me?"**:
   - Look up the file path across every `modules/M*.md` `code_map` table.
   - If the file appears in any module's `code_map` → owned (record which module ID).
   - If the file does NOT appear in any `code_map` → **orphan**.
3. **Triage orphans** — each orphan is one of:
   - **Legitimate exclusion missed in Phase 0**: it should be in `excluded` (e.g., a `.snap` file, a fixture). Add to `excluded` and to `inventories/generated-and-vendor.md` if applicable.
   - **Real coverage gap**: it should belong to a module but no module claimed it. **Self-heal**: trigger `references/self-healing-protocol.md` to either add the file's range to an existing module's `code_map`, or create a new module if it's a distinct responsibility.
   - **Honest unknown**: a file whose role is genuinely unclear; record as `unknown` in COVERAGE.md AND as a row in RISKS.md (severity at least `medium` — unknown code in a project is operational risk).
4. **For files marked `partially covered`** — list the uncovered line ranges. For non-trivial uncovered ranges (>20 lines), apply the same triage as orphans.

### 6.3 Reverse validation completeness gate

Phase 6 is complete only when:

- Every first-party file is `covered` | `partially covered` (with explicit uncovered ranges) | `excluded` (with reason) | `unknown` (with RISKS.md entry).
- The total number of `unknown` files is recorded in `progress.json.stats.unknown_files`.
- The reverse-validation pass left no triage TODOs — every orphan was resolved into one of the four classifications.

If the orchestrator finds it cannot resolve all orphans (e.g., too many to triage in this session), the leftover orphans MUST be recorded in `progress.json.next_frontier` with one bullet each. Mapping cannot be declared complete with un-triaged orphans.

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

For non-trivial repos, **parallel subagent dispatch is the default**, not an optional optimization. A purely-sequential MAPPING run on a 50k+ LOC repo is wasteful and may not finish in a session window. The protocol's correctness assumes parallelism; the orchestrator's job is to coordinate it safely.

### What subagents can and cannot write (refined)

The old "subagents must not write to `.architecture/` directly" was overly broad and caused serial bottlenecking. The refined rule is **per-file-owner**:

**Subagents MAY write directly to per-leaf, non-conflicting files:**

- `modules/M####-<slug>.md` — exactly ONE writer subagent (or batched-writer) writes each leaf's file. Two subagents never write the same leaf path.
- `decisions/<YYYY-MM-DD-HHMM>-<slug>.md` — only the subagent that made the decision writes it.

The orchestrator assigns these files at dispatch time; no two subagents are ever told to write the same path. There is no "merge conflict" because there is no shared file.

**Subagents MUST NOT write to shared / global / index files:**

- `MANIFEST.md` (concurrent writes corrupt the tree)
- `COVERAGE.md` (aggregation owned by orchestrator)
- `RISKS.md` (aggregation owned by orchestrator; subagent contributions flow through their MODULE.md `## Unknowns And Risks` section)
- `README.md` (single coherent author)
- `.meta/progress.json` (orchestrator-only)
- `.meta/preflight.json` (orchestrator-only, written once)
- `interactions/*.md` (Phase 5 stitching aggregates across modules)
- `inventories/*.md` (Phase 2.5 — each inventory file is owned by one subagent though)

If a subagent has findings that need to land in a shared file, it RETURNS them in its report; the orchestrator integrates.

**Subagents MUST NOT edit source code.** Their job is read + report.

### Phase-by-phase parallelism opportunities

Use this table to plan dispatch — not "is parallelism appropriate?" but "which sub-tasks can run together?":

| Phase | Parallel unit | Notes |
|---|---|---|
| 0 (state) | sequential | one orchestrator decision |
| 1 (census) | up to 4 parallel: read top-level docs / map tree shape / build LOC stats / mine git log | each subagent returns structured data, orchestrator assembles `preflight.json` |
| 1.4 (initial RISKS) | sequential | orchestrator-only (writes shared file) |
| 2a (top-level cut) | sequential | one orchestrator decision; produces decision record + initial MANIFEST tree row |
| 2b (full tree planning) | parallel by top-level module — N subagents (one per round-2 module), each explores its subtree and proposes child IDs | orchestrator merges proposals into MANIFEST tree |
| 2.5 (inventories) | parallel — 4 subagents (entrypoints / tests / external-interfaces / generated-and-vendor), each writes its own inventory file | inventories are 4 separate files, no overlap |
| 3 (leaf-criteria verification) | parallel by top-level module — N subagents | each validates its own subtree's leaf candidates |
| 4 (batched writer) | parallel — multiple batches of 5–10 cohesive leaves; one writer subagent per batch produces all docs in the batch | concurrency budget below |
| 4.4 (quality gate) | mechanical check, can run in parallel across leaves but is cheap so sequence is fine | orchestrator |
| 5 (interactions) | parallel — 4 subagents (runtime / data-flow / build-and-config / external-boundaries), each writes its own map | aggregated edges from MODULE.md docs |
| 6.1 (forward coverage) | sequential aggregation | orchestrator |
| 6.2 (reverse coverage) | parallel orphan triage (one subagent per orphan batch, e.g., 10 orphans per batch) | each subagent classifies its batch |
| 7 (risk audit) | sequential aggregation | orchestrator scans every MODULE.md `## Unknowns And Risks` |

### Concurrency budget

Real-world Agent platforms have parallel-tool-call limits. Use these defaults:

- **N = number of round-2 top-level modules** as the unit of parallel dispatch for tree-shaped phases (2b, 3).
- For Phase 4 batched writers (see "Phase 4 leaf-doc strategy" above): launch **up to 6 batches concurrently** (each batch handles 5–10 cohesive leaves). If the platform supports fewer parallel tool calls, throttle. If more, raising the cap is OK.
- For Phase 2.5 (inventories) and Phase 5 (interactions), launch all 4 subagents at once — small and bounded.
- For Phase 6.2 (orphan triage), batch orphans 10 per subagent; cap at 4 concurrent batches.

### Subagent dispatch payload (what each gets)

Every subagent dispatch includes:
- The principles in SKILL.md (or full SKILL.md content).
- The relevant `references/*.md` file(s) for its phase.
- The relevant `templates/*.md` file(s) for its output.
- `preflight.json` (so it knows project context).
- Its **assignment**: subtree path, ID range, file(s) to write.
- The orchestrator's coordination promises: "you own writes to {these files}; you do not write to {these shared files}; report your findings for shared files in the return."

For smaller repos (say, < 5k LOC, < 10 leaves), do everything in one session — the parallelism overhead exceeds the savings.

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
