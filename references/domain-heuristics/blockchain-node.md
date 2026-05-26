# Blockchain Node Heuristics

For full-node software: consensus participants, validators, archive nodes, light clients, sequencers, fork-aware clients.

## When this applies

Concrete signals (need ≥ 2):

- `consensus`, `tendermint`, `libp2p`, `geth`, `besu`, `bor`, `erigon`, `lighthouse`, `prysm`, `cometbft`, `substrate-runtime` in package manifests
- `genesis.json` / `chain_id` / `chainspec.json` / `genesis.ssz` in repo
- A directory named like `consensus/`, `mempool/`, `txpool/`, `engine/`, `forkchoice/`, `vm/`, `evm/`, `wasm/`, `eth/`, `p2p/`
- Build artifacts named like `*-node`, `*d` (daemon), `geth-like` cli
- `RPC` server with chain methods (`eth_*`, `net_*`, `web3_*`, `cosmos.bank.v1.Msg*`)

If only 0–1 of these match, this is probably **smart-contract project** (see `smart-contract.md`) or a chain-adjacent tool (indexer, wallet, bridge), not a full node.

## Typical top-level modules

A well-organized full-node project commonly has these top-level concerns. Names vary; the responsibilities are stable.

| Typical module | One-line responsibility |
|---|---|
| `consensus` | Block proposal + voting + finality rules (BFT/PoS) OR difficulty/PoW |
| `p2p` | Peer discovery, gossip, sync protocols, peer scoring |
| `mempool` (or `txpool`) | Pending tx admission, eviction, ordering for proposers |
| `state` (or `world-state` / `chain-state`) | Account state tree, state pruning, witness generation |
| `storage` (or `db`) | DB engine wrapper, snapshot/restore, schema |
| `rpc` (or `api`) | JSON-RPC / gRPC / REST server exposing chain to external clients |
| `vm` (or `evm` / `wasm` / `runtime`) | Smart-contract execution engine + gas accounting |
| `sync` | Initial chain sync (full / fast / snap / light), reorg handling |
| `crypto` | Hashing, signatures, key derivation, BLS aggregation if applicable |
| `wallet` / `keys` | Local key storage, signing service (sometimes separate binary) |
| `governance` / `staking` | On-chain governance, validator set, rewards/slashing |
| `engine-api` (PoS clients) | Execution-layer ↔ consensus-layer interface |
| `observability` / `metrics` | Prometheus / OTel / pprof endpoints |
| `cli` / `cmd` | Node operator CLI: start, init, keys, query |
| `genesis` / `chainspec` | Genesis loading, chainspec parsing, fork schedule |
| `tools` / `devnet` | Local devnet bootstrap, testnet helpers (sometimes excluded as ops) |

Not every node has all of these. PoW nodes lack `staking`; light clients lack `mempool` and have a thin `state`; sequencer rollups have a thinner `consensus` and a heavy `da-batcher`.

## Common boundary pitfalls

| Pitfall | Why it's wrong |
|---|---|
| Collapsing `consensus` + `engine-api` into one module | Engine API is a network-protocol boundary; consensus rules are pure logic. They evolve independently. |
| Putting `mempool` under `consensus` | Mempool is per-node, consensus is across-node. Different lifecycle, different state ownership. |
| Splitting `state` and `storage` differently than the project does | Some projects use IAVL/MPT (logical state separate from key-value store); some use a single layer. Follow the project's actual seam, don't force a generic split. |
| Treating `vm` as part of `consensus` | VM execution is state-transition; consensus is "which block wins". Conflating these makes upgrades (fork-time VM swaps) hard to reason about. |
| Lumping all `crypto` together | Hash functions, signature schemes, and accumulators may have different audit posture. Subdivide if one is hand-rolled. |
| Hiding `genesis` under `consensus` | Genesis affects deploy / network bootstrap / observability; it spans more than consensus. Keep it visible. |
| Missing a `forkchoice` module | In PoS clients, fork-choice is a distinct algorithm separate from block validation. Forgetting it produces a wiki that can't describe reorgs. |

## Naming conventions

- Use the project's existing terminology when it has strong precedent (e.g., `el`/`cl` in Ethereum PoS clients, `abci` in Cosmos chains).
- Prefer noun phrases over verb phrases: `block-validator`, not `validate-block`.
- For fork-choice specifically: `fork-choice` or `forkchoice` (without space) — both common.
- Distinguish `validator` (a node role) from `block-validator` (a code module).

## Risk hotspots — almost always need `RISKS.md` entries

These areas in a blockchain node are inherently high-blast-radius. Every leaf module in these areas should appear at least once in `RISKS.md`:

- Hand-rolled cryptographic primitives (BLS, KZG, custom signature schemes, custom hashes).
- Consensus algorithm implementations (any divergence from the spec is a hard fork).
- State storage migration code (schema changes risk data loss / state corruption).
- Signing service / key management (compromise = funds loss).
- Hard-fork schedule logic (off-by-one block height = chain split).
- Gas accounting / fee market logic (under-pricing = DoS).
- P2P message validation (malformed message handling = DoS surface).
- RPC method authorization (admin RPC exposure = node takeover).

The protocol's mandatory cross-check (`analysis-protocol.md` Phase 4.2) is especially important on these. Do not skip review of these even in `mapping_mode: "speed"`.
