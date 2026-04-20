# Stellar Capabilities: Overview

**Status:** scaffolded — 2026-04-18
**Purpose:** bounded, relevance-filtered inventory of what Stellar offers that a self-custodial AI-agent wallet can build on.
**Template:** `research/stellar-capabilities/_template.md`

---

## 1. Scope

This folder documents **what the wallet can use**, not Stellar in general. Each capability entry answers four questions:

1. What is it?
2. Why does it matter for an AI-agent wallet (actors, non-negotiables, threats)?
3. What quirks does a programmatic caller hit?
4. What requirements does this imply?

Out of scope for this folder: Stellar consensus, validator operations, Core internals, general protocol theory, historical protocol evolution except where a breaking change is still on the roadmap. Those exist elsewhere and are not differentiating for this wallet.

## 2. File map

| File | Area | Covers (intended scope; to be populated via the template) |
|---|---|---|
| `01-accounts.md` | Accounts and keys | G-accounts, C-accounts (smart accounts), muxed accounts, derivation paths (SEP-05 / BIP-44), reserves and sponsorship, account merge |
| `02-classic-ops.md` | Classic on-chain operations | Payments, path payments, trustlines, multisig and thresholds, set options, fee bumps, sequence-number management, change trust |
| `03-soroban.md` | Soroban | Contract invocation, `SorobanAuthorizationEntry`, events, simulation / preflight, resource fees, footprint / TTL / restore, relevant host functions |
| `04-smart-accounts.md` | Account abstraction | OpenZeppelin stellar-contracts account implementation, custom policy contracts, passkey-based accounts, fee-payer models, upgradeability patterns |
| `05-seps.md` | Ecosystem proposals | Relevance-filtered SEP table (SEP-01, -05, -07, -10, -12, -24, -06, -29, -30, -31, -38, -45, others as relevant). Not a full SEP index — only what an agent wallet plausibly touches. |
| `06-tooling.md` | Tooling and APIs | `stellar-cli` (current scope and commands), Horizon, Soroban RPC, maintained SDKs (Flutter, iOS, PHP, KMP + reference JS/Rust/Java/Python), friendbot, explorers, sandbox options |
| `07-dex-amm.md` | DEX, AMM, liquidity | Classic DEX (manage offers, offer book), classic liquidity pools, Soroban AMMs (Soroswap, Aquarius, Blend, Phoenix, others), cross-venue routing, slippage handling, oracle availability (Reflector etc.) |
| `08-infra-ops.md` | Operational infrastructure | Sponsored reserves, fee-bump strategy, sequence pools and channels, LaunchTube, retry / idempotency semantics, rate limiting on public endpoints |
| `09-soroban-defi.md` | Soroban DeFi beyond DEX/AMM | Lending (Blend), index/portfolio (DeFindex), cross-chain bridges (Axelar, Allbridge, Chainlink CCIP), stablecoin protocols (USDC, USDT, USDT0, EURC, EURAU, PYUSD), staking / perps / yield aggregators. Complements `07-dex-amm.md`. |
| `10-mpp.md` | Machine Payments Protocol on Stellar | MPP (HTTP-402 agentic-payment protocol from Tempo and Stripe; IETF individual submission `draft-ryan-httpauth-payment`): Stellar charge mode via Soroban SAC `transfer` + auth entries; channel mode via one-way payment-channel contract with off-chain cumulative commitments; MCP transport binding; coexistence with x402. |

## 3. How entries are used

Each entry produces candidate requirements in its §8. Those candidates flow into `analysis/03-requirements.md` via the candidates queue. During review, candidates are promoted to MUST / SHOULD / NICE with the capability file path + anchor as the source, or rejected with a recorded reason.

The template's non-negotiable-check is not applicable here (these are native capabilities, not external products). The actor/threat mapping in §3 of each file is what we use to judge relevance.

## 4. Suggested working order

Entries may be researched in any order; subagents work in parallel against stubs. A suggested sequence when single-threaded:

1. `01-accounts.md` — foundational; account types touch every other area.
2. `02-classic-ops.md` and `03-soroban.md` — in parallel; largely independent.
3. `04-smart-accounts.md` — depends on `03`.
4. `05-seps.md` — broad; many items independent of each other.
5. `06-tooling.md` — references the previous five.
6. `07-dex-amm.md` — references `02`, `03`, `06`.
7. `08-infra-ops.md` — references everything; synthesis of operational concerns.

## 5. Relationship to SDKs we maintain

Our own SDKs (Flutter, iOS, PHP, KMP) are the canonical reference for Stellar client-side behaviour when docs and SDKs disagree. Entries cite SDK files when needed, using the path relative to each SDK repo root. SDK repo locations are in `~/.claude/CLAUDE.md`.
