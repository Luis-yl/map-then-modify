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

### 1.3 Build inventories

Create one inventory file per category under `inventories/` using `templates/inventory.md`:

- **`entrypoints.md`** — `main`, `index`, `app`, server bootstrap, CLI definitions, contract `constructor`s, scheduled job registration.
- **`tests.md`** — test framework(s), test directories, test commands from CI/Makefile.
- **`external-interfaces.md`** — HTTP routes, gRPC services, RPC methods, CLI subcommands, contract ABI methods, event signatures, exported library symbols.
- **`generated-and-vendor.md`** — generated code paths + generator command/source; vendored dirs + the upstream they vendor.

### 1.4 Evidence required for everything

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

## Phase 2 — First module cut

Create a first-pass module tree in `.architecture/MANIFEST.md` using `templates/manifest.md`.

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

---

## Phase 3 — Recursive decomposition

For each non-leaf module, in turn:

1. **Re-anchor on code**: enumerate the files and symbols belonging to this module. Use `rg -n` to map cross-file references. Read entry-point files of the node fully.
2. **Identify seams**: where does this node have natural sub-responsibilities?
3. **Apply split triggers** (below) — decide whether to subdivide.
4. **Subdivide**: assign child IDs, write child module docs as hypotheses, recurse.
5. **Or finalize as leaf**: apply leaf criteria, write the full module doc per `templates/module.md`.

**There is no depth limit.** Recurse until every leaf passes leaf criteria.

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

### Progress tracking & resume

Maintain `.architecture/.meta/progress.json`:

```json
{
  "complete": false,
  "started_at": "<ISO8601>",
  "updated_at": "<ISO8601>",
  "next_id": 47,
  "queue": [
    {"id": "M0003-consensus-pow", "state": "in_progress", "parent": "M0003-consensus", "depth": 2},
    {"id": "M0007-p2p", "state": "pending", "parent": null, "depth": 0}
  ],
  "stats": {"modules_total": 0, "leaves_done": 0, "files_read": 0}
}
```

Update after each module. On resume: treat any `in_progress` as interrupted — redo that node from scratch (idempotent re-write).

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

## Phase 7 — Risk audit

Update `.architecture/RISKS.md` per `templates/risks.md` with:

- Low-confidence modules.
- Inferred behavior not verified by tests.
- Dynamic dispatch / reflection / plugin loaders that hide static structure.
- Generated code whose generator source is unclear or missing.
- External systems not available for verification (testnets, oracles, third-party APIs).
- Circular dependencies.
- Broad edit ranges (single anchor spans > ~500 LOC).
- Missing tests on critical modules (consensus, money, auth, upgrade paths).
- Stale docs (line ranges already out of sync at the time of writing).

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

---

## Inter-module reference integrity (final pass)

After all leaf modules are written:

1. For every `outbound` entry across all modules, verify the target module exists in `MANIFEST.md`.
2. For every target, verify the reciprocal entry exists in the target's `inbound`.
3. Any asymmetry = a bug. Fix it.
4. Modules with neither inbound nor outbound and not an entrypoint → suspect dead code. Flag in `RISKS.md` rather than deleting analysis.

---

## Stopping condition

Mark `progress.json` as `"complete": true` only after **all** of:

- `.architecture/README.md` written (from `templates/architecture-readme.md`).
- `.architecture/MANIFEST.md` written and lists every module ID with status.
- `.architecture/COVERAGE.md` classifies every first-party file.
- `.architecture/RISKS.md` records all known uncertainty.
- `modules/M*-*.md` written for every module in the tree.
- `interactions/*.md` written for the 4 standard maps.
- `inventories/*.md` written for entrypoints, tests, external interfaces, generated/vendor.
- Analysis run record written in `analysis-runs/`.
- All inbound/outbound references resolve symmetrically.
- No `in_progress` entries remain in `progress.json`.

If the repo is too large to finish in one session, leave a coherent partial wiki with explicit frontier items in `COVERAGE.md` and `RISKS.md`, and mark `"complete": false` with a clear `next_frontier` queue.

---

## Final report to the user

≤ 10 lines:
- Total modules mapped (leaves / non-leaves).
- Wiki location: `.architecture/`.
- Coverage summary (covered / partial / unknown / excluded counts).
- Top risks flagged.
- Hint: "Next time you ask me to modify or extend this project, I'll use this wiki to keep changes safe."

Then stop. Do not propose changes. Do not start development mode unprompted.
