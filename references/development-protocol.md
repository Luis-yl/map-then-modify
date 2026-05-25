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

## Phase 7 — Verify

Run verification in layers:

1. **Targeted** — tests for changed leaf modules.
2. **Affected parents** — tests for parent modules whose behavior may have changed.
3. **Interaction-edge tests** — integration or smoke tests for changed interactions.
4. **Broader suite** — when blast radius is high (see `boundary-and-evidence-rules.md` blast-radius rules).

If a verification command fails:
- Determine whether the failure is related to the change.
- Investigate related failures — never claim success when relevant verification is failing.
- For unrelated environmental failures (e.g. flaky CI test unrelated to your area), record them in the dev plan's `## Open risks` and inform the user.

Never mark a step complete while its verification is failing.

---

## Phase 8 — Update architecture docs

Update `.architecture/` to reflect the new state. Mandatory whenever the change touched:

- **Responsibility** of a module → update Summary, Owns/Does Not Own.
- **Code ranges** (lines shifted, files split, new files) → update `code_map`.
- **Interactions** (new caller, dropped dependency, signature change) → update Inbound/Outbound + the relevant `interactions/*.md`.
- **State ownership** → update State And Data Ownership.
- **Invariants** → update Invariants.
- **Failure modes** → update Failure Modes.
- **Tests** → update Tests And Verification + `inventories/tests.md`.
- **External behavior** (route, ABI, public surface) → update `inventories/external-interfaces.md`.
- **Generated artifacts** → update `inventories/generated-and-vendor.md`.
- **Risks** → update or close entries in `RISKS.md`.

Bump `last_verified` on every touched module doc.

If the change added a brand-new module, write its `modules/M*.md` from scratch using `templates/module.md`, add it to `MANIFEST.md`, and update `ARCHITECTURE.md` (`README.md` in `.architecture/`) if the addition is structurally significant.

If the change deleted a module (rare), set `status: deprecated` in its doc (don't delete the file — it preserves history), prune `MANIFEST.md`'s "Module Index" row, scrub references in other modules' inbound/outbound lists.

If docs did **not** change (because the change was purely behavioral and didn't touch anything the wiki tracks), state that explicitly in the final report.

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
Write its `modules/M*.md` as part of phase 8 with all sections populated. Assign the next stable ID. Do not defer — the next request will need it.

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
