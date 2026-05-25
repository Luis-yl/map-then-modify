# Self-Healing Protocol

Use this protocol when analysis or development discovers code that is relevant to the task but missing from `.architecture/`, or when the wiki contradicts the implementation.

**Core stance**: undocumented related code is **not** a reason to ask the user for technical confirmation. It is a reason to analyze, document, expand the plan, and continue. The user does not know this codebase; the skill owns these decisions.

This protocol is the precise procedure behind principle P4 in `SKILL.md`.

---

## Purpose

The architecture wiki must heal as work discovers reality. Static analysis is never complete on a real codebase — dynamic dispatch, reflection, conditional imports, generated code, runtime plugins, and config-driven behavior all hide structure that only surfaces under specific tasks.

When a dev task surfaces something the wiki missed, the right move is to:
1. Stop expanding the edit (do not silently absorb the new area).
2. Map the area enough to make the current change safe.
3. Update the wiki so the next task benefits.
4. Update the active dev plan's allowed surface.
5. Continue.

This trades a small amount of in-task time for permanent wiki improvement.

---

## Triggers

Run self-healing when **any** of the following happens during analysis or development:

### Triggers during development
- Code outside the planned edit surface must be touched.
- An unlisted caller or callee affects the change.
- An unlisted config controls relevant behavior.
- A test fixture defines undocumented behavior the change interacts with.
- Generated code or generator inputs are missing from the wiki.
- A `code_map` line range is stale (anchor no longer at the listed range).
- A module doc contradicts implementation (e.g., doc says "pure function, no side effects" but code writes to disk).
- A test failure points to a module not in the impact set.
- Runtime wiring differs from the manifest.
- A module's public interface is broader than documented.

### Triggers during mapping — leaf-criteria self-check fail

When writing a leaf module's MODULE.md (Phase 4), if **any** of the following becomes true, the planned leaf is actually a non-leaf and must be subdivided. These triggers are deliberately operational — "I felt unsure" is not enough; one of these objective conditions must be cited as the trigger.

- The Summary cannot be written in 2 sentences without using `AND` / `OR` / `plus` / `; also` to enumerate features.
- The `code_map` has more than ~10 rows whose `Role` values represent **different business concerns** (not just different stages of one workflow). A leaf with 12 rows that are all stages of one DI bootstrap is fine; a leaf with 12 rows that are 12 unrelated HTTP endpoints is not.
- Different `code_map` rows have fundamentally different `Edit Guidance` for unrelated reasons (some pure CRUD, others protocol-critical; some safe, others requires-careful-context for incompatible reasons).
- The module has separate inbound interaction patterns serving separate parts of itself (e.g., admin-session auth for one subset, node-Bearer auth for another subset).
- Removing any single `code_map` row would leave a coherent module behind — strong sign of separable responsibility.
- The module's `Public Surface` cannot be summarized in one "what does this module expose" sentence without enumerating multiple unrelated surfaces.

If any trigger fires:

1. Pause the leaf doc.
2. Mark the original ID `status: split` in MANIFEST + write its (now-non-leaf) overview doc that delegates to children.
3. Assign next stable IDs to children using the parent's headroom block (`progress.json.next_id`, incrementing).
4. Update MANIFEST.md `Module Tree` + `Module Index` + relevant `Concern Index` row.
5. Write child MODULE.md docs (each may itself be a leaf or recurse further).
6. Update relevant `interactions/*.md` edges (edges that pointed at the parent now point at the appropriate child).
7. Write `self-healing/YYYY-MM-DD-HHMM-leaf-split-<old-id>.md` recording the split + rationale.
8. Resume — writing the children's docs, not the original.

---

## Procedure

Execute in order:

### Step 1 — Stop and isolate

Stop editing code outside the documented surface. If you have partial in-flight edits:
- If the edits are coherent (build/compile), keep them. Pause new edits.
- If the edits leave the tree in a broken state, decide: revert this step's edits (preferred for a clean self-heal) OR finish this one minimal change before pausing.

Do not start new edits in the divergent area until the wiki covers it.

### Step 2 — Identify and classify

Identify the discovered area and why it matters to the current task. Classify it as one of:

- **New module** — represents responsibilities not yet captured anywhere.
- **Child module** — sub-responsibility of an existing module, needs splitting.
- **Interaction edge** — connects two existing modules but the edge isn't in any interaction map.
- **Generated artifact** — output of a generator that needs to be documented.
- **Test contract** — behavioral specification hidden in a test fixture.
- **Config boundary** — runtime behavior driven by a config value that wasn't tracked.
- **External boundary** — interaction with a system not previously cataloged.
- **Stale range** — the area is documented but at the wrong line range.
- **Misplaced responsibility** — the wiki says module A owns this, but it's actually owned by module B.

### Step 3 — Analyze the area (scope-controlled)

Inspect enough code to understand the area's relation to the planned change. Scope rule:

- **Document the minimal slice needed for the current change to be safe.**
- If the discovered area is small, fully document it.
- If the discovered area is large (e.g., you stumbled into an entire undocumented subsystem):
  - Document the slice that interacts with the current change.
  - Mark the rest with `status: partial` and `confidence: medium-or-low`.
  - Add a `RISKS.md` entry noting the unmapped portion.
  - Continue only if the planned change can be bounded safely without mapping the full subsystem.

### Step 4 — Update or create module docs

Based on the classification:

- **New / child module**: assign the next stable ID, write a full doc using `templates/module.md`. Add to `MANIFEST.md`. If a child, update the parent's "Owns" / child-list accordingly.
- **Stale range / misplaced responsibility**: update the existing `code_map` entry. Bump `last_verified`.
- **Interaction edge**: update inbound/outbound on both endpoints, and add the edge to the relevant `interactions/*.md`.
- **Generated artifact / config / external boundary**: update the relevant inventory file.
- **Test contract**: update the target module's Tests And Verification section, and `inventories/tests.md`.

Every update must include evidence (per `boundary-and-evidence-rules.md`).

### Step 5 — Update COVERAGE.md and RISKS.md

- **COVERAGE.md**: if a file went from `unknown` to `covered`/`partial`, update it. If a file's coverage status changes (e.g., a range was claimed by a new module), update it.
- **RISKS.md**: add a new risk entry if confidence is still low after the heal, or if external systems are involved and not verifiable.

### Step 6 — Write the self-healing report

Write `.architecture/self-healing/YYYY-MM-DD-HHMM-<trigger-slug>.md` using `templates/self-healing-report.md`. Mandatory contents:

- What was missing (or stale or contradictory).
- Why it matters to the current task.
- Evidence collected (file:lines).
- Wiki updates made (which `.architecture/` files were touched).
- New allowed edit surface (added ranges).
- Remaining risk after the heal.
- Decision: continue or block? Why?

### Step 7 — Update the active dev plan

Edit the active `dev-plans/*.md` in place:

- Add a `## Heal events` section (if absent) and append an entry for this heal.
- Update the `Allowed Edit Surface` table with new ranges.
- Update the `Forbidden Nearby Areas` if the heal moved boundaries.
- Add to the `Architecture Read Set` (the newly-read docs).
- Add to `Open Risks` if the heal left any.

### Step 8 — Inform the user (briefly, factually)

In the chat, 1–2 sentences. Template:

> Discovered related code not covered by `.architecture/`: `<short description>`. I analyzed `<file:lines>`, updated `<which docs>`, expanded the dev plan, and continued within the new documented boundary.

If the heal was substantial (multiple new modules, significant scope expansion), be slightly longer but no more than 4–5 lines. **Do not** ask the user to approve the heal.

### Step 9 — Continue

Resume the dev plan from where it was paused.

---

## When to pause and ask the user (rare)

Self-healing itself does not require user input. But the heal may *reveal* a question only the user can answer. Pause and ask **only** when:

- The requested product behavior is ambiguous given the newly discovered constraints.
- The safe implementation requires a destructive data operation (e.g., dropping a column, replaying state).
- Credentials or private external systems are required to verify.
- There are now mutually exclusive acceptable behaviors.
- The change would intentionally alter a public / security / protocol contract not specified by the original request.

When you pause:
- State the question crisply.
- Offer 2–3 concrete options with their tradeoffs (use AskUserQuestion).
- Describe the heal already done so the user knows where you are.

After the user answers, resume.

---

## Scope control — when the discovered area is too large

If the heal would balloon into mapping an entire undocumented subsystem, you have three options. Pick based on the current task:

1. **Slice + risk** (default): map only the slice that intersects the current change. Add a `RISKS.md` entry for the unmapped portion. Continue.
2. **Pause + scoped sub-mapping**: if even the slice is unsafe without more context, pause the dev task and run a focused mapping pass on the affected area (a partial Phase 1–4 over just that subtree). Document in an `analysis-runs/` entry. Then resume dev.
3. **Block + escalate**: if the heal reveals the change cannot be made safely (e.g., the current request implicitly depends on a system the wiki shows is too risky to touch), block, write a `decisions/YYYY-MM-DD-HHMM-<slug>.md` entry using `templates/decision-record.md` explaining why, and surface to the user.

Default to option 1. Move to 2 or 3 only when 1 leaves residual unsafe surface.

---

## Quality bar for a self-healing update

A self-healing update is acceptable only when it includes:

- What was missing (concrete area, not vague).
- Why it matters (concrete impact on the current task).
- Exact code evidence (file paths + line ranges + anchors).
- Updated module or interaction docs (which files, which sections).
- Coverage status delta (what moved from `unknown` to `covered`, etc.).
- Confidence assessment on the new docs.
- Impact on the active dev plan (allowed surface delta, new risks).

Vague self-healing notes ("noticed some related stuff, added it to the wiki") are worse than no heal. They give false confidence. **Re-do the heal** if the report doesn't meet this bar.

---

## What self-healing is NOT

- Not a justification for unbounded scope creep. If every change triggers heroic re-mapping, the original mapping was too shallow — go back and deepen it.
- Not an excuse to skip Phase 1 (initial mapping). Heal complements mapping; it does not replace it.
- Not a way to "fix" the wiki by deleting inconvenient claims. Update them, don't erase them. Old claims with `status: superseded` preserve history.

---

## After many heals — quality signal

If a single dev task triggers more than ~3 heal events, that is a signal the initial mapping for this area was too coarse. Note this in `RISKS.md` (or in `MANIFEST.md`'s `Next Frontier` section) so the next analysis pass can deepen it.

This is healthy feedback, not a failure. The skill is designed to discover this gap exactly here, rather than at the time the cascade-damage bug ships.
