# Development Protocol

Use this protocol when the user requests a change in a codebase that has already been mapped (or where a focused mapping pass will be done first).

**Goal**: Make the requested change while preventing accidental edits to undocumented modules or hidden dependencies. The architecture wiki is **not** passive documentation — it is the edit control system.

The user does not know the codebase. **Do not ask** technical questions. Make the safe choice and inform.

---

## Phase 0 — Intake

Classify the user's request:

- single-module change,
- multi-module feature,
- bug fix,
- refactor,
- integration / external dependency,
- test-only change,
- docs-only change,
- unknown scope.

If scope is unknown, use the architecture wiki to discover likely modules. **Do not ask the user to identify code modules.**

Restate the request in 2–4 sentences. Confirm the **what**, not the **how**. The wiki dictates the how.

Legitimate questions to ask the user here (per P5 in SKILL.md):
- Product behavior tradeoffs ("should the fee apply to existing positions or only new ones?")
- Mutually exclusive requirements
- Destructive operations
- Accepting a known risk

Do **not** ask:
- "Which file should I modify?"
- "Should this cascade to module X?"
- "Is this code related?"

---

## Phase 1 — Load architecture context (only what's needed)

Read, in order:

1. `.architecture/MANIFEST.md` — find candidate modules by name, concern index, and module tree.
2. `.architecture/COVERAGE.md` — check whether the candidate areas are `covered` / `partial` / `unknown`.
3. `.architecture/RISKS.md` — read entries touching the candidate areas.
4. Target module docs (one or more `modules/M*.md`).
5. Parent module docs (for architectural context).
6. Child module docs if the target is not a leaf (you may need to drill in).
7. Relevant interaction maps (`interactions/runtime.md` etc.) — for blast-radius reasoning.
8. Relevant inventories (e.g. `entrypoints.md` if changing CLI behavior; `external-interfaces.md` if changing public API).

**Do not** load the entire `.architecture/` tree. Only what relates to the request. The user is paying for both correctness and speed.

Build two sets:
- **Primary set**: modules whose code you expect to modify.
- **Adjacent set**: modules in the primary set's inbound/outbound. You won't modify these — you must respect their published contracts.

---

## Phase 1.5 — When the request does not map to any module

If Phase 1 returns **no candidate modules** in `MANIFEST.md` for the request, **do not** proceed to Phase 2 and **do not** ask the user "where is this in the code?". Instead, classify which of three scenarios applies:

### Scenario A — Feature exists in code but is missing from the wiki

**Signal**: The request mentions a concept that plausibly exists. A targeted `rg`/`grep` over the codebase for the concept's keywords finds matching code.

**Procedure**:
1. Run targeted discovery: `rg -i '<keyword1>|<keyword2>'`, `rg -l '<function_or_type_pattern>'`, follow imports.
2. Trigger **SELF-HEALING** (`references/self-healing-protocol.md`) on the discovered area:
   - Classify what was missed (new leaf module / child of existing module / interaction edge / etc.).
   - Write or update `modules/M####-<slug>.md` for the affected area.
   - Update `MANIFEST.md` (Module Tree, Module Index, Concern Index), `COVERAGE.md`, `interactions/*.md`.
   - Write a `self-healing/YYYY-MM-DD-HHMM-<slug>.md` report.
3. Inform the user in 1–2 sentences ("Discovered `<concept>` lives in `<file>` (was unmapped). Added M####-<slug> to the wiki. Continuing.").
4. Resume Phase 1 with the now-updated wiki. Then continue to Phase 2.

### Scenario B — Feature does not exist; the request is to build it

**Signal**: Targeted discovery finds **no** matching code, but the concept is semantically coherent with the project's scope (per `MANIFEST.md`'s Module Tree and Concern Index — the project is the kind of project where this feature *would* fit).

**Procedure**:
1. Confirm the concept does not exist by:
   - `rg -i` with multiple keyword variants — empty.
   - Checking `inventories/external-interfaces.md` and `inventories/entrypoints.md` — no related entry.
2. Decide architectural placement of the new feature:
   - **Standalone new module**: the feature has a distinct responsibility — plan a new `M####-<slug>`. Reserve its ID in `.meta/progress.json`'s `next_id`.
   - **Extension of an existing module**: the feature is an additional capability of an existing module — extend that module's owned ranges in its `code_map` post-implementation.
3. Continue to Phase 2 with the chosen placement.
4. In Phase 3 (dev plan), the **Allowed Edit Surface** will include "new file(s)" rows (for new modules) or extended ranges of existing modules. The **Target Modules** table will mark the new module as `M####-<slug>` (proposed).
5. In Phase 8 (post-implementation), creating `modules/M####-<slug>.md` (full doc per `templates/module.md`) and updating `MANIFEST.md` is **mandatory** — not optional.

This is not self-healing — there is nothing to "heal" because no code exists yet. This is normal development that happens to create a new module.

### Scenario C — Request is outside the project's scope

**Signal**: Targeted discovery finds nothing AND the concept is semantically incompatible with the project's scope (e.g., user asks for "PoW difficulty tuning" in a project whose `MANIFEST.md` lists only PoS consensus modules; user asks for "GraphQL endpoint" in a project that has no HTTP server at all).

**Procedure**:
1. **Pause** development. Do not write a dev plan.
2. Tell the user concretely what you found (or didn't find) and why the request seems mismatched. Cite specific modules from `MANIFEST.md` that establish the project's scope.
3. Offer 2–3 concrete reformulations using `AskUserQuestion`:
   - The closest semantic equivalent within the project's actual scope.
   - A clarification of intent ("did you mean X?").
   - A literal interpretation that would require massive new infrastructure ("strictly building this would require adding modules A, B, C — confirm you want that scope?").
4. Wait for the user's answer. Then re-enter Phase 0 with the resolved request.

This is the only branch from Phase 1.5 that **blocks on the user**. Scenarios A and B are autonomous.

### Decision rule (summary)

```
no MANIFEST hit
    │
    ▼
targeted rg / grep for the concept
    │
    ├── finds code               →  Scenario A: SELF-HEAL, resume
    │
    └── finds nothing
         │
         ├── concept fits project →  Scenario B: new-module dev, autonomous
         │     scope
         │
         └── concept conflicts    →  Scenario C: pause + ask user
               with project scope
```

**Do not** collapse B and C. New-feature work in B is autonomous (the user wants it built). C must pause because the request is likely a misunderstanding and proceeding would burn time on the wrong target.

---

## Phase 2 — Focused refresh

Before planning edits:

1. **Verify target files still exist**: `ls` each file in the target module's code map.
2. **Refresh line ranges**: re-`rg` each anchor in the code map. If the anchor moved, update the range *now* before the dev plan freezes it.
3. **Check recent changes**: `git log --oneline -20 -- <target files>` — anything surprising? Anything that suggests a different module owns this area now?
4. **Check whether tests still exist**: each entry in the module's `tests` section.
5. **Check coverage staleness**: if `COVERAGE.md` says this area is partial or stale, self-heal before planning.

If any check reveals divergence, branch into self-healing immediately. Do not proceed with planning on a stale wiki.

---

## Phase 3 — Write the execution plan

Create `.architecture/dev-plans/YYYY-MM-DD-HHMM-<request-slug>.md` per `templates/dev-plan.md`.

The plan MUST include:

- **User request** (verbatim or precise restate).
- **Architecture read set** — which docs were loaded.
- **Target modules** — IDs + role in this change + confidence.
- **Allowed edit surface** — file + line range / anchor + module ID for every place you are authorized to touch.
- **Forbidden nearby areas** — adjacent code that may look related but must not be touched without self-healing.
- **Expected interaction changes** — which edges in the interaction maps will change.
- **Self-healing triggers** — what to watch for during execution that would force a self-heal.
- **Implementation steps** — small, ordered, each pointing at one allowed-surface entry.
- **Verification plan** — targeted tests, then broader, with the actual commands.
- **Rollback notes** — how to revert if the change must be undone.
- **Open risks** — anything you'd flag to a senior reviewer.

The plan is **not** a permission request. It is a self-contract. Show it to the user in the chat (so they can redirect if they disagree about product intent), then **execute it directly** — do not pause for technical sign-off.

If the plan involves obviously high-stakes changes (payment flows, key management, irreversible migrations, consensus/protocol changes), surface that risk inline above the plan and pause briefly for the user to react. This is rare.

---

## Phase 4 — Establish verification

Before modifying any production code, identify how you will verify the change. Preferred order:

1. Existing focused tests for the target module.
2. Characterization tests for current behavior (write them if they don't exist and the change risks regression).
3. New tests for the new behavior.
4. Integration tests for affected interactions.
5. Build / type-check / lint.
6. Manual smoke command (only if no executable check exists).
7. Static reasoning — recorded as risk when nothing executable is available.

For behavior changes, add or update tests when the project has a test pattern for the target module. For projects with no viable tests, record the limitation in the dev plan AND in `RISKS.md`.

---

## Phase 5 — Implement inside boundaries

For each step in the plan, in order:

### Before each edit
- Confirm the target file + line range is inside an allowed-surface entry.
- Re-read ~30 lines around the edit for local invariants.

### During each edit
- Make the smallest change that achieves the step.
- Preserve existing public contracts unless the plan says otherwise.
- Avoid opportunistic cleanup outside the target range.
- **If you encounter code that looks relevant but is not in any allowed-surface entry → DIVERGENCE**. Stop the edit. Self-heal. Then resume.

### After each edit
- If lines shifted significantly, note the drift — you'll refresh `code_map` ranges in phase 8.
- If the change altered an interaction edge, note it.
- Run the cheapest relevant test if available. Don't yet run the full suite.

If a needed edit falls outside the allowed surface, **stop and self-heal**. Do not ask the user for technical confirmation. Do not silently expand the edit.

---

## Phase 6 — Handle undocumented dependencies (self-healing trigger)

This is the core anti-cascade-damage moment. The triggers and full procedure are in `references/self-healing-protocol.md`. Summary of triggers during development:

- An unlisted caller of the target module must be updated.
- A serializer/validator outside the module controls behavior.
- A test reveals a hidden fixture contract.
- Config wiring outside the module controls runtime behavior.
- Generated code must be regenerated from an unlisted source.
- A test failure points to a module not in the impact set.
- Outbound dependency's signature differs from what the wiki says.

When any trigger fires:
1. Stop expanding the edit.
2. Load `references/self-healing-protocol.md` and follow its procedure.
3. Update the dev plan in place (add a `## Heal events` section noting what changed and why).
4. Inform the user in 1–2 sentences.
5. Resume from where you paused.

---

## Phase 7 — Verify (tests + impact oracle)

Run verification in layers:

1. **Targeted** — tests for changed leaf modules.
2. **Affected parents** — tests for parent modules whose behavior may have changed.
3. **Interaction-edge tests** — integration or smoke tests for changed interactions.
4. **Broader suite** — when blast radius is high (see `boundary-and-evidence-rules.md` blast-radius rules).
5. **Impact oracle cross-check** (when available — see below).

If a verification command fails:
- Determine whether the failure is related to the change.
- Investigate related failures — never claim success when relevant verification is failing.
- For unrelated environmental failures (e.g. flaky CI test unrelated to your area), record them in the dev plan's `## Open risks` and inform the user.

Never mark a step complete while its verification is failing.

### Phase 7.5 — Impact oracle cross-check (optional external provider)

After Phase 5 edits are made (i.e., there is a real diff against the base commit) and before declaring Phase 7 verification done, run an **impact oracle** to catch callers / dependencies the dev plan may have missed. The oracle is an **independent sensor**, not a tie-breaker — its findings feed into triage, not into automatic edits.

#### Provider detection

The skill supports any provider that produces an impact set in the schema below. The reference provider is **Understand-Anything (UA)**: detected via `.understand-anything/knowledge-graph.json` presence and a working `/understand-diff` command. Other providers (tree-sitter scripts, LSP-based scanners, custom AST tools) can fill the same role; the protocol treats them all uniformly via a normalized JSON output.

If no provider is detected, Phase 7.5 is **skipped silently** — record `impact_oracle: "none"` in the dev plan's `Execution Log` and continue. The base verification layers (1–4) remain the primary verification path.

#### Provider invocation contract

The provider is invoked with the current diff (staged + unstaged + new files) and must return a normalized impact set:

```json
{
  "provider": "ua-knowledge-graph | tree-sitter-cli | lsp | custom",
  "source_commit": "<git SHA the impact was computed against>",
  "computed_at": "<ISO8601>",
  "affected_nodes": [
    {
      "id": "<provider-native ID, e.g., function:src/foo.ts:bar>",
      "file": "<repo-relative path>",
      "line_range": "<start-end or 'full'>",
      "symbol": "<symbol name if applicable>",
      "kind": "<file | function | class | route | event | ...>",
      "reason": "<modified | reverse-dependency | forward-dependency | test-of>"
    }
  ]
}
```

Provider stdout / file path → orchestrator reads → caches at `.architecture/.meta/impact-oracle/<dev-plan-slug>.json` for audit.

#### Three-way comparison (quorum)

The orchestrator now has three sources for "what's affected":

1. **Plan-declared** — modules in the dev plan's `Target Modules` + `Allowed Edit Surface`.
2. **Wiki-derived** — every module whose `Inbound Interactions` or `Outbound Interactions` lists a touched file (resolved via MTM `code_map` crosswalk: `provider node file:line → MTM module ID`).
3. **Oracle-found** — `affected_nodes` from the provider's response, resolved through the same crosswalk.

Classify every affected node into one of three buckets:

| Bucket | Definition | Meaning |
|---|---|---|
| `confirmed` | In oracle AND in wiki AND in plan | Expected impact, high confidence |
| `wiki-only` | In wiki Inbound/Outbound but NOT in oracle | Oracle missed (likely dynamic / config / event / runtime-only), OR wiki is overstating dependency. **Do not act on this without re-inspection.** Possible self-heal candidate if oracle is right and wiki is stale. |
| `ua-only` (or `oracle-only`) | In oracle but NOT in wiki | Either oracle false positive, OR a real caller wiki missed (cascade-damage candidate). **Always triage before acting.** |

#### Triage rules for `oracle-only` (the cascade-damage probe)

For each `oracle-only` node, run mechanical triage before deciding whether to self-heal:

1. **Type-only import?** Resolve the node's evidence file:line. If the only reference is a `type` / `interface` import (TypeScript) or a build-time-only reference, mark `triage: type-only`, skip self-heal.
2. **Barrel re-export?** If the file is an index/barrel that just re-exports symbols, mark `triage: barrel`, skip self-heal.
3. **Dead code?** If the node is in a file with no inbound from any reachable entrypoint, mark `triage: unreachable`, skip self-heal but record as suspected dead code in `RISKS.md`.
4. **Test helper?** If the node is in a test file that is not part of the project's protected behavior surface, mark `triage: test-helper`, skip self-heal.
5. **Generated / vendored?** Check against `inventories/generated-and-vendor.md`. If yes, mark `triage: non-source`, skip self-heal (and instead address the generator if behavior actually changed).
6. **None of the above?** This is a **real cascade-damage candidate** — mark `triage: cascade-candidate` and trigger **SELF-HEALING** (see `references/self-healing-protocol.md`). The self-heal must (a) add the missing inbound row to the wiki module that owns this caller, (b) re-evaluate whether the dev plan's Allowed Edit Surface needs to expand, and (c) re-run any verification affected by the surface change.

Triage results are recorded in the cached oracle artifact under `triage_results`. The protocol does NOT skip Phase 7.5 just because triage is tedious — the triage is exactly the mechanism that turns oracle noise into actionable signal.

#### What Phase 7.5 does NOT do

- **It does not gate verification on oracle silence.** A green oracle (zero `oracle-only`) is good evidence, not proof. Other failure modes (dynamic dispatch, config-driven routing, event subscribers) remain invisible to static oracles. Verification layers 1–4 (the tests) remain primary.
- **It does not automatically expand the edit surface.** Only `triage: cascade-candidate` triggers self-heal, and the self-heal goes through the normal protocol (analyze → update wiki → expand plan → resume edits).
- **It is not a replacement for thorough Phase 1 module-doc Inbound/Outbound coverage.** A wiki-only-correct project would pass Phase 7.5 silently; a wiki-with-gaps project benefits because the oracle catches what the writer missed.

#### Crosswalk reliability note

The crosswalk `provider node file:line → MTM module ID` uses the wiki's `code_map` table. Failure modes (per `boundary-and-evidence-rules.md` §10):

- A file may appear in multiple modules' `code_map` (range-level ownership) — resolve by line range overlap.
- A node's call site may live in a caller function whose own line range is in module A, while the node it references is in module B — the inbound edge goes from A's perspective. The crosswalk must distinguish "where is this node" (target file:line) from "who is calling it" (caller's enclosing range).
- Shared utility files: `coverage ≠ owner`. A util function used by 5 modules may have inbound rows in each of those modules' docs; oracle-only checks should match against any of them.

---

## Phase 8 — Mandatory Wiki Sync

**A dev task is NOT complete until this phase passes.** Code edits, tests, and successful verification are necessary but not sufficient — the wiki and the codebase must be in a consistent state at the end of every dev task. A stale wiki at the end of a task is a regression that misleads the next dev task in this area.

This is enforced by the completion gate at the end of this phase. Do not report the task complete unless every relevant item is done.

### 8.1 Wiki sync actions, by change type

For each shape of change in this dev task, execute ALL listed sync actions. Multiple change types in one task → execute all matching rows.

| Change type | Required wiki sync actions |
|---|---|
| Modified existing module's code (no contract change) | Update affected `modules/M####-<slug>.md`: refresh `code_map` line ranges, bump `last_verified` (date + SHA), add/adjust `Evidence Log` rows if symbols changed. |
| Changed a module's public surface (signature, route shape, event payload, ABI) | Update module's `Public Surface` table. For every inbound caller (per module's `Inbound Interactions`), update the corresponding `Outbound Interactions` row on the calling module. Update the matching `interactions/*.md` edge. If the surface is externally visible: update `inventories/external-interfaces.md`. |
| Added a new function / route / event / job to existing module | Add row to module's `code_map`. If exported, add to `Public Surface`. If a new route or RPC method: add to `inventories/external-interfaces.md`. If a new entrypoint (CLI / daemon / cron / scheduled job): add to `inventories/entrypoints.md`. |
| Added a new test | Update module's `Tests And Verification` table. Add row to `inventories/tests.md`. |
| **Added a new module** (a new responsibility boundary not in the existing tree) | (a) Read `progress.json.next_id` for the next stable ID; increment for the new module. (b) Write a full `modules/M####-<slug>.md` from `templates/module.md` — all required sections per `analysis-protocol.md` Phase 4, ending with the END-OF-MODULE sentinel. (c) Add a row to `MANIFEST.md` `Module Index`. Insert into `MANIFEST.md` `Module Tree` at the correct parent. Update `MANIFEST.md` `Concern Index` row(s) that this module participates in. (d) Update parent module's `Scope.Owns` / `Scope.Does Not Own` / child subtree map. (e) Add all relevant edges to `interactions/*.md` (runtime / data-flow / build-and-config / external-boundaries as applicable). (f) Update relevant `inventories/*.md`. (g) Update `progress.json` `next_id` and `stats.modules_total` / `stats.leaves_done` / `stats.non_leaves`. |
| **Removed a module** | (a) Set `status: deprecated` in the module's MODULE.md — do NOT delete the file (it preserves history and old dev-plan references). (b) Update `MANIFEST.md` `Module Index` row to show `status: deprecated`. Remove from `Module Tree` rendering (keep the ID reserved — never reuse). (c) Scrub references in every other module's `Inbound Interactions` / `Outbound Interactions`. (d) Update interaction maps. (e) Update inventories. (f) Update `progress.json` stats. |
| **Renamed a module** (responsibility unchanged, label change only) | Keep the stable ID. Change only the `title` field in module frontmatter + `MANIFEST.md` `Module Index` title column. Do NOT change directory name, slug, or ID. |
| Module **split** (one leaf became multiple children) | Parent doc: set `status: split`, replace body with redirect pointers to children. Children: assign next stable IDs (each), write full MODULE.md each. Update MANIFEST tree, parent's child list, interaction edges. |
| Module **merged** (two modules collapsed into one) | Surviving module absorbs. Other module(s): set `status: merged`, body becomes a redirect pointer. Do NOT delete merged docs. Scrub references in interaction maps and other modules. |
| Changed an interaction edge (added / removed / signature changed) | Update both endpoints' `Inbound` / `Outbound` tables AND the corresponding `interactions/*.md` edge-list row. |
| Changed a state ownership claim | Update module's `State And Data Ownership`. Update `interactions/data-flow.md`. |
| Changed an invariant | Update module's `Invariants` section. If the change *weakens* an existing invariant (relaxing a constraint that other modules depend on), record a row in `RISKS.md`. |
| Changed external / network / storage boundary | Update `inventories/external-interfaces.md` and `interactions/external-boundaries.md`. |
| Generated code's generator changed | Update `inventories/generated-and-vendor.md`. Document the new generator command. Regenerate artifacts via the documented command. |
| Discovered a new project-level risk during this change | Add a row to `RISKS.md` Active Risks. Reference the source module in the `Area` column. |
| Behavioral change that does not touch any tracked aspect above | Bump `last_verified` on the touched module(s). State explicitly in the final report that no other wiki sync was required and why. |

### 8.2 Completion gate (checklist)

Before reporting the task complete to the user, verify EVERY item below. If any item is false, the task is not done — either complete it, or explicitly mark the deferral as a new entry in `RISKS.md` with reason.

- [ ] Every touched module's `last_verified` field is bumped to today's date AND the current commit SHA.
- [ ] Every new module has a full `MODULE.md` (not a stub) per `templates/module.md`, ending with the END-OF-MODULE sentinel.
- [ ] `MANIFEST.md` `Module Tree`, `Module Index`, and `Concern Index` reflect every add / remove / rename / split / merge from this task.
- [ ] Every affected interaction edge is updated in the relevant `interactions/*.md`.
- [ ] Every affected inventory row is updated in `inventories/*.md`.
- [ ] `progress.json` `next_id`, `stats.modules_total`, `stats.leaves_done`, `stats.non_leaves` reflect new totals.
- [ ] No module's `Inbound` / `Outbound` table references a deleted or non-existent module ID.
- [ ] Every row in any touched module's `## Unknowns And Risks` either appears in `RISKS.md` or is explicitly resolved by this task.
- [ ] If the task added a new external boundary (HTTP route, RPC method, event topic, generated artifact, persistence schema), it appears in the correct `inventories/*.md` AND `interactions/external-boundaries.md` where applicable.
- [ ] No `modules/M*.md` file lacks the END-OF-MODULE sentinel (signals interrupted-mid-write).

### 8.3 If no wiki sync is required

If the change is purely behavioral and touched nothing the wiki tracks (e.g., an internal optimization with no observable contract change), state this explicitly in the final report:

> "No wiki sync was required for this task because the change touched only internal-implementation aspects not tracked by the wiki (specifically: [name what was touched])."

This statement is itself a sync artifact — it tells future readers that the absence of wiki edits was deliberate, not forgotten.

---

## Phase 9 — Final report

To the user, in ≤ 10 lines:
- What changed (1–2 sentences, plain language).
- Dev plan path.
- Modules touched (by ID, not file paths — IDs are stable).
- Wiki updates (which `.architecture/*.md` files were edited).
- Self-healing performed (if any) — what was missing, what was added.
- Verification commands run + status (pass / fail / skipped with reason).
- Remaining risks (briefly).

**Do not** include a closing "what I learned" or "next steps" unless the user asks.

---

## Blocking conditions

Block (pause and ask) **only** when progress requires something not inferable from the repo:

- Missing credentials or external access.
- Unavailable external system required for verification.
- Destructive data operation.
- Product behavior ambiguity (genuinely two reasonable choices).
- Security or compliance decision (e.g. accepting a known vulnerability).
- Mutually exclusive requirements.

**Do not block** merely because the wiki is incomplete. Self-heal it.

---

## Edge cases

### Request touches modules across many subtrees
If `primary_modules` > ~5, the change is cross-cutting. Group edits by module in the plan and verify each group's contract integrity after the group is complete (not just at the end). Cross-module test failures discovered late are the worst to debug.

### Request hits a `gotchas` or `risks` warning
Surface the warning in the plan **before** executing: "Module X has documented gotcha: 'do not modify Y without Z'. Your request requires modifying Y. I will [strategy]." Then proceed. The user retains the ability to redirect by responding to the chat.

### Wiki is older than ~50 commits since last analysis
Compute: `git rev-list <max(last_verified_sha across loaded modules)>..HEAD --count`. If > 50, warn the user that the wiki may be stale, offer to refresh affected areas. Do not force a full re-mapping.

### Request is trivial ("just rename this variable")
Follow the spine but compress. One-section plan, one edit, run tests. Don't over-engineer for trivial changes — do follow the spine.

### Request requires a new module
Follow the **"Added a new module"** row in Phase 8.1 — full MODULE.md + MANIFEST tree/index/concern updates + parent's Owns/Does Not Own + interaction edges + inventory entries + `progress.json` `next_id` + `stats` updates are all mandatory. Do not defer — the next dev task in this area will need a complete wiki.

### Generated/vendored code is involved
See `boundary-and-evidence-rules.md` for the generated/vendor rules. Edit the generator source, not the artifact, unless the repo clearly treats the artifact as source.

---

## Success criteria

After phase 2 completes:
- Code change is correct, minimal, bounded.
- No unrelated module was touched.
- All verification commands relevant to the change ran and passed (or failures are recorded).
- Wiki accurately reflects the new state.
- The next dev request, days or weeks later, can use the wiki as if nothing happened.
