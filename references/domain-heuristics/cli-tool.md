# CLI Tool Heuristics

For terminal-first programs: developer tools, ops scripts, file processors, batch tools, shell utilities. Not for daemons that happen to expose a CLI (those are usually web-saas or library + thin CLI).

## When this applies

Concrete signals (need ≥ 2):

- `entry_points_candidates` dominated by `cmd/*/main.go`, `bin/*`, `src/cli/*`, `__main__.py`.
- A command framework in deps: `cobra` (Go), `clap` (Rust), `click` / `typer` (Python), `commander` / `yargs` / `oclif` (Node).
- Top-level binary named the tool (e.g., `git`, `kubectl`, `aws`, `npm`).
- No long-running server, no HTTP handler.
- README is "install, run X, see output Y" — not "deploy and access via browser".

If the tool also exposes a server mode (`tool serve`), still classify as CLI — the server is an internal mode, not the primary surface. Use `web-saas` only if the server mode dominates the codebase.

## Typical top-level modules

| Typical module | One-line responsibility |
|---|---|
| `cli-entry` (or `main`) | Top-level binary: parses argv, dispatches to commands |
| `commands` | One sub-module per command (or per command group) — the "verbs" the tool exposes |
| `config` | Config file parsing (`.toolrc`, `~/.config/tool/`, env vars) — single source of truth |
| `runtime-context` | Per-invocation state shared across commands: logger, output formatter, current working dir, profile selection |
| `output` | Rendering: terminal table / JSON / YAML / progress bars / colors / TTY detection |
| `input` | Stdin handling, prompt helpers (interactive flows), credential prompts |
| `core-logic` | The domain operations the CLI commands wrap (often: a library that the CLI is a thin facade over) |
| `state` (if applicable) | Local state files: lock files, cache, history, sessions (e.g., `~/.cache/tool/`) |
| `integrations` | External systems the tool talks to (cloud APIs, package registries, SSH, git, docker) |
| `plugins` | Plugin loader if applicable (extension points exposed by the tool) |
| `update` (or `self-update`) | Tool-update logic (`tool update` / version check) |
| `telemetry` | Usage reporting (often opt-in, sensitive area) |
| `tests` / `fixtures` | Shared test infra, golden output files |

For a small CLI, this collapses to: `cli-entry` + `commands` + `core-logic` + `config`. For something like `kubectl`/`aws-cli`, you'll have 15+ command sub-modules.

## Common boundary pitfalls

| Pitfall | Why it's wrong |
|---|---|
| One module per command, even for trivial commands | A `version` command and a `help` command are not separate modules. Group cohesive commands. Aim for "command groups" not "command files". |
| Lumping all commands under one `commands` leaf | Same as web-saas: each command group is a real boundary. Each group has its own external dependencies, its own failure modes. |
| Putting `config` parsing inside `cli-entry` | Config has multiple sources (file, env, flags), validation rules, defaults. A separate module makes it clear and testable. |
| Treating `output` rendering as part of each command | If JSON / YAML / table rendering is shared, it's a separate module. Each command should only emit "data", not "rendered output". |
| Forgetting `state` when the tool has local cache/lock files | State files are persistence — they should be tracked with the same rigor as a DB module in web-saas. Concurrent invocations, lock files, crash recovery. |
| Mixing `telemetry` into commands | Telemetry has privacy posture (opt-in, what fields, retention). Always a distinct module with its own RISKS.md entries. |
| Treating `update` / `self-update` as part of `cli-entry` | Self-update is sensitive (privilege escalation, binary replacement, signature verification). Always distinct. |

## Naming conventions

- `commands` (plural) for the collection; individual command groups can be `commands/auth`, `commands/deploy`, etc., or sibling modules `auth-commands` / `deploy-commands`.
- Don't name a module after a CLI flag or option (`verbose-mode-handler` is wrong; output verbosity is a `runtime-context` concern).
- `runtime-context` is the right name for "stuff passed through every command" — alternatives: `app-state`, `context`, `session`.

## Risk hotspots — almost always need `RISKS.md` entries

- `self-update` logic (signature verification, atomic replacement, downgrade attacks).
- Credential storage (keyring access, plaintext fallback, file permissions on `~/.config/tool/credentials`).
- Shell-out and command construction (shell injection risks).
- File system operations on user-supplied paths (path traversal).
- Lock files (deadlock on crash, stale lock recovery).
- Telemetry: PII leakage, opt-out adherence.
- Network calls from `core-logic` to external services (cert validation, timeout defaults).
- Plugin loading (arbitrary code execution from `~/.config/tool/plugins/`).
- Cross-platform behavior gaps (Windows path handling vs Unix, locale issues).

## Notes specific to this domain

- CLI tools often have **two distinct user surfaces**: the human-facing TTY mode (with prompts, colors, progress) and the script-facing pipe mode (JSON / quiet output). These may not be separate modules but `Modification Guide` should mention them per-command.
- **Argument parsing libraries' framework conventions matter** — `cobra` puts subcommands in directories, `clap` uses derive macros. Module boundaries should follow the framework's grain (don't fight cobra's `cmd/` convention to enforce a "logical" boundary).
- Many CLI tools are **thin wrappers over a library** in the same repo. If so, the library is `core-logic` and CLI is a top-level module that calls into it. Don't merge them — they often have different release cadences (CLI bumps for UX, library bumps for behavior).
- **Local state files** are easy to overlook because they're not in the repo. Inventory them in `inventories/external-interfaces.md` under "Local filesystem state" — `~/.config/tool/`, `~/.cache/tool/`, lock files in CWD.
