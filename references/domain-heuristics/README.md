# Domain Heuristics

Per-domain "what's a sensible top-level cut?" guidance, loaded conditionally during Phase 2a.

These files are **references, not commands**. They describe the *typical* shape of well-organized projects in each domain so the orchestrator can recognize patterns and propose more accurate top-level modules — especially in domains the AI has not deeply seen before. They are **suggestions to compare your cut against**, not templates to copy.

If a heuristic file says "blockchain nodes usually have a `consensus` top-level module" and the project you're analyzing genuinely doesn't (e.g., a wallet-only client that delegates consensus), **trust the code, not the heuristic**. The criteria in `analysis-protocol.md` Phase 2a still govern.

## When to load

After Phase 0/1 preflight completes, the orchestrator inspects `preflight.json` for signals of project domain. Load the matching heuristic file(s) as additional context for Phase 2a (top-level cut):

| Signal in `preflight.json` | Load |
|---|---|
| `languages` contains `solidity` + `foundry.toml` / `hardhat.config.*` in `package_managers` | `smart-contract.md` |
| top-level files include `genesis.json` / `consensus.go` / `tendermint`/`libp2p` in deps / chain naming in `entry_points_candidates` | `blockchain-node.md` |
| `entry_points_candidates` contain Fastify / Express / Next.js / Django / Rails / Flask / FastAPI server bootstrap | `web-saas.md` |
| `package_managers` shows the project IS a library (e.g., `main` / `exports` field in package.json without an `app` entrypoint) — no server runtime | `library-sdk.md` |
| `entry_points_candidates` are dominated by `cmd/*` / `bin/*` / cli command registration AND no long-running server | `cli-tool.md` |

A project can match multiple — load all that apply (e.g., a chain that also ships a CLI: load both `blockchain-node.md` + `cli-tool.md`).

A project matching NONE of these (research code, scientific computing, ML pipelines, mobile apps, embedded firmware, game engines, etc.) — do not invent a heuristic file. Use Phase 2a criteria directly. Note the missing heuristic in `progress.json.next_frontier` as a candidate for the library to grow.

## How heuristic files are structured

Each heuristic file contains:

1. **When this applies** — concrete signals that confirm the domain match
2. **Typical top-level modules** — a list of common module candidates with a one-line responsibility for each
3. **Common boundary pitfalls** — splits that look natural but are wrong, and vice versa
4. **Naming conventions** — slugs that are conventional in this domain (so future readers recognize them)
5. **Risk hotspots** — modules in this domain that always warrant `RISKS.md` entries

## How to use (rule)

When loading a heuristic for Phase 2a:

1. Read the file fully.
2. Compare the heuristic's "typical top-level modules" against what you'd cut from the source alone.
3. For every typical module in the heuristic, ask: "does this project have this responsibility? if yes, do my proposed top-level modules capture it?". If not, you may have missed a top-level seam — investigate before finalizing.
4. For every top-level module YOU propose that the heuristic does NOT have, justify it briefly in the `decisions/<...>-top-level-cut.md` decision record. This forces "non-standard" cuts to be deliberate, not accidental.

Heuristics are checks against laziness, not architecture police. A justified deviation is fine. A drift-by-default is not.
