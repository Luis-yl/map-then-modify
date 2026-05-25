# Boundary And Evidence Rules

These rules apply whenever creating, updating, or relying on `.architecture/` docs, and whenever editing code under this skill. They are the precise rules behind the high-level principles in `SKILL.md`.

---

## 1. Evidence is mandatory

Every architectural claim must be traceable to at least one of:

- file path + line range,
- symbol name (function, class, type, trait, method, component),
- route or command registration site,
- test name,
- config key,
- package manifest entry,
- migration file,
- generated artifact + its generator source,
- runtime command output (cite the command and the relevant excerpt),
- documented external interface (RFC, protocol spec, ABI declaration).

A claim like "probably handles validation" is not allowed unless marked explicitly as inference with `confidence: low` and recorded as a risk.

When citing evidence in a module doc's Evidence Log, use the format:

```
| Claim | Evidence |
| --- | --- |
| Validates block header before storing | `consensus/header.go:42-118` (function `VerifyHeader`) |
| Emits `block.committed` event after persist | `state/store.go:201` + test `state/store_test.go:TestEventEmit` |
```

---

## 2. Confidence levels

Every claim — module status, code-map entry, interaction edge, invariant, failure mode — carries a `confidence` field:

| Level | Meaning |
|---|---|
| `high` | Traced through implementation AND verified by tests, runtime wiring, or strong static evidence. |
| `medium` | Traced through implementation, but runtime behavior or tests not fully verified. |
| `low` | Inferred from names, structure, partial references, or incomplete evidence. |
| `unknown` | Not enough evidence to make a claim — but the claim is needed structurally. Record what's missing. |

Hidden uncertainty causes cascading damage. Recorded uncertainty does not.

Default to the highest level you can prove. When in doubt, downgrade — `medium` honestly held is more useful than `high` falsely held.

---

## 3. Module boundaries are range-level, not file-level

A single file can contain multiple modules. A module owns **ranges**, not necessarily whole files.

Each `code_map` entry must include:

| Field | Required | Notes |
|---|---|---|
| `file` | yes | repo-relative path |
| `line_range` | yes | `start-end` (1-indexed, inclusive) |
| `anchor` | yes | symbol/route/config-key/test-name that survives line drift |
| `role` | yes | one-line: what this range contributes to the module |
| `edit_guidance` | yes | safe / requires-careful-context / requires-self-healing-first |
| `confidence` | yes | high / medium / low / unknown |
| `last_verified` | yes | YYYY-MM-DD (the date the anchor was confirmed at this range) |

Line ranges are snapshots. They drift as the file changes. **Always refresh ranges by re-grepping anchors before editing.**

---

## 4. Stable anchors

Prefer anchors that survive line drift, in this priority order:

1. Function / class / type / trait / method / component name.
2. Route name / command name / config key.
3. ABI or schema name.
4. Test name (the actual `test_foo` identifier).
5. Exported symbol.
6. Unique comment text — only when no code-level symbol is available, and only if the comment is identity-bearing (e.g., a TODO ID, ticket reference).

If no stable anchor exists at all (very rare — typically pure data files, generated blobs), record that as a risk in `RISKS.md` with severity `medium` minimum.

---

## 5. Allowed edit surface

During development, the allowed edit surface is the **union** of:

- Ranges listed in the active dev plan's `Allowed Edit Surface` table.
- Ranges added by a self-heal pass within this same dev plan.
- Tests or docs explicitly named in the plan.

**Anything else is out of scope** until a self-healing pass documents it.

---

## 6. Unknown / undocumented related code

"Related code" includes anything that:

- Calls the target module,
- is called by the target module,
- owns data read or written by the target module,
- generates target code,
- validates target outputs,
- serializes or deserializes target data,
- controls feature flags or runtime configuration affecting the target,
- has tests that define target behavior,
- handles errors from the target module,
- exposes target behavior externally.

If related code is not in `.architecture/`, **run self-healing before editing it** (see `self-healing-protocol.md`).

---

## 7. Generated and vendored code

**Do not edit generated or vendored code** unless the repo clearly treats it as source (rare; e.g., committed lock files that are meant to be edited by hand are themselves not "source" — they're snapshots; treat them as such).

When generated code is involved in a change:

1. Identify the generator (script, codegen tool, compiler).
2. Document generator input and output in `inventories/generated-and-vendor.md`.
3. Edit the source of generation, not the artifact.
4. Regenerate if the project supports it (cite the command).
5. Document the artifact's relationship to its source.

If the generator cannot be found, record the risk in `RISKS.md` and stop the edit there. Modifying an artifact without knowing the generator is a guaranteed regression.

---

## 8. Interaction edge types

Use these types consistently across module docs and interaction maps:

| Type | Meaning |
|---|---|
| `call` | Direct function / method / module call. |
| `event` | Event bus, callback, hook, signal, observer, pub/sub. |
| `data` | Reads or writes a shared data structure or store. |
| `config` | Behavior controlled by a configuration value. |
| `build` | Build-time or generation dependency. |
| `network` | HTTP, RPC, p2p, websocket, queue, or external protocol. |
| `storage` | DB, filesystem, object storage, chain state, cache. |
| `test` | Tests or fixtures defining expected behavior. |
| `runtime` | Process lifecycle, scheduler, worker, daemon, plugin loader. |

An edge can have multiple types if it genuinely combines them (e.g., a function call that also persists state could be `call+storage`). Prefer a single primary type when possible — clearer for blast-radius reasoning.

---

## 9. Blast radius

A change has **broader blast radius** when it touches:

- Public API shape (HTTP/gRPC/JSON-RPC routes, exported library symbols, ABI).
- Persistent schema (DB tables, on-chain storage slots, file formats).
- Serialization format (wire format, encoding).
- Consensus / protocol logic.
- Authentication or authorization.
- Money, tokens, balances, billing, or any irreversible state.
- Build or deployment configuration.
- Shared utility used by many modules.
- Generated-code source.
- Error handling crossing module boundaries.

Broader blast radius requires:
- Broader verification (run integration / smoke / full suite, not just targeted tests).
- Updated interaction maps.
- Explicit `## Open risks` entries in the dev plan.
- Potentially: a separate sign-off step where the user is informed inline before the edit ships.

Narrower blast radius (purely internal refactor of one function, with all callers in one module) requires only targeted verification.

---

## 10. Module IDs — stability rules

Module IDs are forever. They are how dev plans, risks, and interaction maps refer to modules across time.

- Format: `M{4-digit-zero-padded}-<kebab-slug>`. Examples: `M0001-rpc-server`, `M0042-pow-difficulty`.
- `M0000-root` is the conceptual root.
- The next ID is tracked in `.architecture/.meta/progress.json` under `next_id`.

**Rules**:

- **Never renumber** existing IDs.
- **Never reuse** a retired ID.
- If a module is **renamed**, preserve the ID and update the title only.
- If a module is **split**, keep the parent ID and create new child IDs. The parent doc stays (with `status: split` and pointers to children) if the parent itself becomes empty.
- If two modules are **merged**, preserve old docs with `status: merged` and a pointer to the surviving module. Do not delete merged docs — they preserve dev-plan references.
- If a module is **deprecated**, set `status: deprecated` and explain in the doc. Do not delete the file.

---

## 11. Staleness signals

Architecture docs are stale when **any** of:

- Line ranges no longer match their anchors.
- Files moved.
- Tests changed expected behavior of the module.
- Configs changed runtime wiring.
- Generated outputs changed.
- A dependency was upgraded.
- The module was edited without a corresponding doc update.

When staleness affects a development request, refresh the relevant docs **before** editing code. Do not edit on a stale wiki.

A doc is "fresh enough" for a dev request when its `last_verified` SHA is `git merge-base HEAD <last_verified_sha>` reachable AND no files in its `code_map` have changed since that SHA (`git diff --name-only <sha>..HEAD -- <files>` is empty).

---

## 12. Minimum documentation for safe edit

Before editing a leaf module, the wiki must identify all of:

- Owned code ranges.
- Non-owned nearby ranges ("Does Not Own").
- Inbound callers.
- Outbound dependencies.
- State or side effects.
- Tests or verification commands.
- Known risks for this area.

If **any** is missing AND relevant to the change, self-heal first.

For non-leaf modules, you typically need to descend to leaves before editing. But if the change is genuinely cross-cutting (e.g., adding a new field to a system-wide event), the non-leaf's interaction maps may be enough — provided the leaf changes are themselves bounded.

---

## 13. Naming module slugs

- Use kebab-case.
- Be domain-specific, not generic: `pow-difficulty-adjustment` not `algo`, `mempool-eviction` not `cleanup`.
- Reflect the module's responsibility, not its location: `block-validator` even if the code lives at `internal/foo/bar.go`.
- Keep it under ~40 characters.
- When in doubt, look at the module's `Summary` and pick 2–4 nouns/adjectives that uniquely identify the responsibility.

---

## 14. Cross-references between docs

When a doc refers to another doc, use the module ID as the canonical reference:

```markdown
Calls `M0017-rpc-router` to dispatch requests.
```

Not `the rpc module` or `the file under rpc/`. IDs survive renames and moves.

Cross-references in tables follow the same rule. The reader can always resolve `M0017-rpc-router` → file path via `MANIFEST.md`'s Module Index.
