# Smart-Contract Project Heuristics

For on-chain contract codebases: token systems, DeFi protocols, governance frameworks, NFT systems, bridges, intent systems. Excludes the off-chain infrastructure that consumes contracts (those go under web-saas or library).

## When this applies

Concrete signals (need ≥ 2):

- `.sol`, `.cairo`, `.move`, `.fc`, `.tact`, `.huff`, `.vy` files
- `foundry.toml`, `hardhat.config.*`, `truffle-config.*`, `remappings.txt`
- `forge`, `hardhat`, `anchor`, `scarb` in build/test commands
- A `contracts/` or `src/` directory dominated by contract code
- `*-deploy.s.sol` / `script/Deploy.*` / `deployments/` / `broadcast/` directory

## Typical top-level modules

| Typical module | One-line responsibility |
|---|---|
| `core` (or `protocol`) | The contract that owns the protocol invariants — what makes this project itself |
| `token` | ERC-20 / 721 / 1155 / token-extension logic, if a token is part of the system |
| `access-control` | Role definitions, permission grants, admin gating |
| `governance` | DAO voting, proposals, timelock, governance token |
| `treasury` / `fee-collector` | Fee accumulation, distribution, withdrawal |
| `oracle-integration` | Adapters for Chainlink / Pyth / Redstone / custom price feeds |
| `upgrade-proxy` (or `proxy`) | UUPS / Transparent / Beacon proxy + impl slot management |
| `vault` / `pool` / `market` | Domain-specific holding-and-accounting logic (varies by protocol type) |
| `interfaces` | External interface declarations (`IERC20`, `IFlashLoanReceiver`, etc.) |
| `libraries` | Pure-function utilities (math, address, encoding) reused across contracts |
| `abi-bindings` | Auto-generated bindings for off-chain consumers (TypeScript / Go / Rust) |
| `deploy-scripts` | Foundry scripts / Hardhat tasks for deploying + initializing |
| `fork-tests` (or `integration-tests`) | Tests that fork mainnet/testnet state (separate from unit tests) |
| `gas-snapshots` | Reference gas measurements, often generated artifacts |

A protocol-light project may collapse to just `core` + `token` + `access-control` + `tests` + `deploy`. A protocol-heavy DeFi project may have 8+ top-level modules.

## Common boundary pitfalls

| Pitfall | Why it's wrong |
|---|---|
| Splitting "contracts" and "interfaces" but treating interfaces as the boundary | Interfaces are declarations; the boundary is the implementation. Consumers usually depend on impl ABIs anyway. |
| Lumping `upgrade-proxy` with `core` | Proxy + impl is a deliberate seam. Mixing them obscures the upgrade story and what's actually deployed. |
| Treating `libraries` as a single leaf | Math libs, address libs, and crypto libs have very different audit posture. Sub-split when relevant. |
| Forgetting `deploy-scripts` | Deploy scripts are part of the security perimeter (constructor args set permissions). Always a top-level module. |
| Hiding `fork-tests` under unit `tests` | Fork tests have different determinism, different CI cost, different external dependencies. Separate. |
| Treating ABI bindings as "build artifacts" only | If bindings are committed and consumed by other modules in the same repo, they're a real interaction edge — must appear in `interactions/build-and-config.md`. |
| Collapsing `oracle-integration` per oracle vendor into "oracles" | Vendor switching is operationally significant — keep per-vendor sub-modules so failures localize. |

## Naming conventions

- Use the protocol's own naming: `vault` not `pool` if the contracts say `IVault`.
- Distinguish `OnlyOwner` style from `AccessControl` role-based — module name should hint at it (`role-control` vs `single-admin`).
- Proxies: `uups-proxy`, `transparent-proxy`, `beacon-proxy` — be specific. "Proxy" alone is ambiguous.
- Tests: `forge-tests` / `hardhat-tests` if the project uses both; otherwise just `tests`.

## Risk hotspots — almost always need `RISKS.md` entries

Smart-contract security failures cost money directly. Every leaf module in these areas should appear at least once in `RISKS.md`:

- Upgrade authority (who can call `upgradeTo`, what timelock guards it).
- Initialize functions and their callable-only-once invariants.
- Oracle staleness assumptions (heartbeat, deviation thresholds).
- Reentrancy guard placement (especially in vaults/pools with hooks).
- Permission-changing functions (granting / revoking roles).
- Constructor / initializer argument validation (foot-gun: pass wrong owner = lost contract).
- Cross-chain message verification (bridge security model).
- Token transfer hooks / callbacks (ERC-777, ERC-1155 onReceived).
- Gas-griefing surfaces in unbounded loops.
- Flash-loan attack vectors (any function that takes a price as input).
- Storage slot collisions in proxies.

The protocol's mandatory cross-check (Phase 4.2) is non-negotiable here. `mapping_mode: "speed"` does not skip review on contract code.

## Notes specific to this domain

- Contract addresses are evidence too. Module docs for deployed-and-verified contracts should cite the on-chain address in Evidence Log when relevant.
- Auditor reports (Trail of Bits, OpenZeppelin, Spearbit, Cantina, etc.) sitting in `audits/` directories are first-class wiki evidence — link to them in Evidence Log when an invariant comes from a finding.
- Fork tests' setup blocks often define implicit invariants ("at block N, pool has X liquidity"). These belong in the relevant pool/vault module's Invariants section.
