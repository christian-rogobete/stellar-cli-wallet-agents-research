# Soroban DeFi (beyond DEX / AMM)

**Status:** in progress
**Last updated:** 2026-04-18
**ID:** 09-soroban-defi

---

## 1. What it is

Everything DeFi-adjacent on Stellar that is not a spot swap venue. `07-dex-amm.md` covers trading; this file covers lending, vault-style index products, cross-chain bridges, and the heterogeneous stablecoin landscape the wallet denominates balances in.

Common thread: an unattended caller holding a bounded budget acts on positions whose *state is not local* — an interest-accruing loan, a rebalancing vault share, a bridge transfer awaiting remote attestation, a stablecoin whose issuer can clawback. Each is a T2/T4/T5 surface the wallet must bound without relying on the agent to understand the semantics.

## 2. Components

URLs accessed 2026-04-18.

### 2.1 Lending — Blend

Blend is an isolated-markets lending protocol on Soroban. Anyone can permissionlessly create a pool with chosen reserves, oracle, and risk parameters; a pool funds a `backstop` module (BLND:USDC 80:20 LP) to activate. Source: [`blend-contracts`](https://github.com/blend-capital/blend-contracts) at commit `895845f`, AGPL-3.0 ([`LICENSE`](https://github.com/blend-capital/blend-contracts/blob/895845f/LICENSE)). Blend v2 passed Code4rena + Certora formal verification Q1 2025 (`code4rena.com/audits/2025-02-blend-v2-audit-certora-formal-verification`, mitigation review April 2025).

- **Interaction surface.** All actions flow through `pool::submit(from, spender, to, requests)` with a `Vec<Request>` of `{request_type: u32, address, amount}` (`pool/src/contract.rs:110`). `RequestType` enum: `Supply`, `Withdraw`, `SupplyCollateral`, `WithdrawCollateral`, `Borrow`, `Repay`, `FillUserLiquidationAuction`, `FillBadDebtAuction`, `FillInterestAuction`, `DeleteLiquidationAuction` (`pool/src/pool/actions.rs:21-50`). No separate `borrow()` entrypoint — intent composes as a request vector.
- **Interest-rate model.** Three-slope curve with reactive modifier (`pool/src/pool/interest.rs:22-106`). Slopes change at the configured target utilisation, again at 95%; utilisation error integrates into a rate modifier (0.1× to 10×). Every reserve touch accrues interest.
- **Oracle dependency.** Each pool pins a SEP-40 `PriceFeedClient` at `ReserveConfig.oracle` (`pool/src/pool/pool.rs:106-131`). Prices cache within a transaction; price older than 24 hours panics `PoolError::StalePrice`. Mainnet pools use Reflector. A stale-price panic aborts a valid repay — T7 where a lagging oracle blocks *de-risking*.
- **Liquidation.** Not a direct call. The liquidator first posts `new_liquidation_auction(user, percent_liquidated)` (`pool/src/contract.rs:204`) opening a Dutch auction; any address fills it via `FillUserLiquidationAuction`. Bad-debt and interest auctions are separate (`pool/src/auctions/`).
- **Mainnet deployments** (`docs.blend.capital/mainnet-deployments`): BLND `CD25MNVTZDL4Y3XBCPCJXGXATV5WUHHOWMYFF4YBEGU5FCPGMYTVG5JY`; emitter `CCOQM6S7ICIUWA225O5PSJWUBEMXGFSSW2PQFO6FP4DQEKMS5DASRGRR`; backstop `CAQQR5SWBXKIGZKPBZDH3KM5GQ5GUTPKB7JAFCINLZBC5WXPJKRG3IM7`; pool factory `CDSYOAVXFY7SM5S64IZPPPYB4GVGGLMQVFREPSQQEZVIWXX5R23G4QSU`. Three tracked pools: Fixed USDC:XLM, DAO-managed (USDC, EURC, XLM, AQUA), fixed-forex CDP (`coinbase.com/price/blend`). `07-dex-amm.md` §2 scoped Blend out of spot swaps; it surfaces here because `A1` rebalancing and `A3` pay-in-X routes through Blend's `Borrow`+swap pattern.

### 2.2 Index / portfolio — DeFindex

DeFindex is a vault protocol: tokenised multi-asset container with pluggable yield strategies. Source: [`defindex`](https://github.com/paltalabs/defindex) at commit `f8b5c61`, GPL-3.0 ([`apps/contracts/vault/Cargo.toml`](https://github.com/paltalabs/defindex/blob/f8b5c61/apps/contracts/vault/Cargo.toml)).

- **Architecture.** `Factory` deploys `Vault` instances; each `Vault` holds `AssetStrategySet`s — one or more assets, each with whitelisted `Strategy` contracts implementing `DeFindexStrategyTrait` (`defindex/apps/contracts/vault/src/interface.rs:8`). Mainnet strategies: fixed-pool (USDC, EURC, XLM) and Blend / Yieldblox variants over USDC, EURC, XLM, CETES, AQUA, USTRY, USDGLO (`docs.defindex.io/contract-deployments/mainnet-deployment`; `messari.io/project/de-findex/profile`).
- **Deposit / withdraw.** `deposit(amounts_desired, amounts_min, from, invest: bool)` returns `(actual_amounts, shares_minted, investments)` (`vault/src/interface.rs:126-132`). `amounts_min` fences share-mint slippage; `invest: true` routes funds into strategies immediately. `withdraw(df_amount, min_amounts_out, from)` burns shares, returns `Vec<i128>` (`:154`).
- **Role and policy model.** Four roles: Emergency Manager, Vault Fee Receiver, Manager, Rebalance Manager (`vault/src/interface.rs:14-19`). `rebalance` is Rebalance-Manager-only (`:403-412`); `pause_strategy`, `unpause_strategy`, `rescue` are Manager-or-Emergency-Manager (`:169-209`); `upgrade(new_wasm_hash)` works unless the vault was constructed with `upgradable: false` (`:63-73`, `:388-400`). A third-party-managed vault is a smart-wallet-shaped trust anchor on top of Soroban contract trust.
- **Agent angle.** User-operated vault (agent *is* manager) fits `A1` / `A2`. Third-party-managed vault is `A3` service-consumer with manager-key risk — T5 because "vault X" can be either shape without the wallet distinguishing.
- **Mainnet.** Factory `CDKFHFJIET3A73A2YN4KV7NSV32S6YGQMUFH3DNJXLBWL4SKEGVRNFKI`; vault impl hash `ae3409…1468b`; strategy impl hash `11329c…5da988` shared across instances.

### 2.3 Cross-chain bridges

Four trust models, not interchangeable.

- **Axelar (universal gateway).** Endorsed by `developers.stellar.org/docs/tools/infra-tools/cross-chain`. Source: `github.com/axelarnetwork/axelar-amplifier-stellar`, Apache-2.0. Soroban contracts: Gateway, Gas Service, Operators, ITS, Upgrader (`github.com/axelarnetwork/axelar-contract-deployments/blob/main/stellar/README.md`). Stellar runs ITS in **Hub mode**: token messages hub through the Axelar amplifier chain. Trust: threshold-signature PoS validator set plus relayers. Stellar joined the Axelar network February 2026 (`cryptonewsnavigator.com/academy/article/stellar-blockchain-development-attracts-40-more-projects-this-year`). Mainnet Soroban addresses are not aggregated at a single canonical URL — see §7.
- **Allbridge Core (stablecoin bridge).** Soroban contracts at `github.com/allbridge-io/allbridge-core-soroban-contracts`, audited by Quarkslab March 2024 (`blog.quarkslab.com/allbridge-core-stellar.html`). Four contracts: Bridge, Messaging, Pool (StableSwap), Gas Oracle. Trust: off-chain relayers plus per-stablecoin liquidity pool per chain — a liquidity-swap with a message, not a validator set. Connects nine chains including Base, OP Mainnet, Arbitrum; USDC is primary.
- **Chainlink CCIP.** Announced for Stellar September 2025 as Scale-program membership (`stellar.org/blog/foundation-news/stellar-to-join-chainlink-scale-and-adopt-data-feeds-data-streams-and-ccip-to-power-next-gen-defi-applications`). CCIP v1.6 live on Solana May 2025. Trust: Chainlink DON plus Risk Management Network. Mainnet Soroban addresses not published 2026-04-18 — integration in progress. T5/T8 exposure grows: Coinbase migrated 7 B USD in wrapped assets to CCIP-only December 2025 (`coindesk.com/web3/2025/12/11/coinbase-taps-chainlink-ccip-as-sole-bridge-for-usd7b-in-wrapped-tokens-across-chains`).
- **USDT0 / LayerZero OFT.** Tether's omnichain USDT over LayerZero OFT (`theblock.co/post/380434/tether-pegged-usdt0-omnichain-stablecoin-passes-50-billion-in-cumulative-transfers`). 15 chains at 2026-04; Stellar not on the list. If routed later, trust becomes LayerZero's DVN set.
- **Not live.** Wormhole has no Soroban contracts 2026-04. Squid Router depends on Axelar — if it routes through Stellar it does so via Axelar ITS, not as an independent contract.

### 2.4 Stablecoin protocols

Stellar's stablecoin surface is **heterogeneous in issuance shape** — T3 at the unit level, T5 at the counterparty level.

- **USDC (Circle).** Classic asset, issuer `GA5ZSEJYB37JRC5AVCIA5MOP4RHTM335X2KGX3IHOJAPP5RE34K4KZVN`. SAC-wrapped on Soroban consumption. Issuer flags: CLAWBACK_ENABLED per CAP-35 ([`stellar-protocol/core/cap-0035.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0035.md)). CCTP V2 not on Stellar; Stellar USDC ↔ EVM USDC routes via Allbridge Core or Axelar ITS.
- **EURC (Circle).** Same issuer shape, clawback-enabled.
- **EURAU.** MiCAR-compliant euro, integrated 13 April 2026 (`coinmarketcap.com/cmc-ai/stellar/latest-updates/`). Flags to verify per-use.
- **USDT (Tether).** Tether does not natively issue on Stellar at 2026-04. Any classic-asset trustline called "USDT" without a verified issuer routes to an unknown counterparty — direct T5 trap. USDT0 (§2.3) not on Stellar.
- **FxDAO synthetics (USDx, EURx, GBPx, FXG).** Classic-asset shape for legacy-wallet compatibility (`fxdao.io/docs/addresses/`). Assets issued from `GAVH5ZWACAY2PHPUG4FL3LHHJIYIHOFPSIUGM2KHK25CJWXHAV6QKDMN`; Vaults `CCUN4RXU5VNDHSF4S4RKV4ZJYMX2YWKOH6L4AKEKVNVDQ7HY5QIAO4UB`; Oracle `CB5OTV4GV24T5USEZHFVYGC3F4A4MPUQ3LN56E76UK2IT7MJ6QXW4TFS`. Licence not asserted on the docs page. CDP behaviour, not issuer-backed.
- **Soroban-native stablecoins.** None have displaced the classic-issuer model in 2026. Pattern is *classic issue + SAC-wrap* — same token-identity ambiguity `07-dex-amm.md` §4 flags.
- **SAC clawback.** CLAWBACK_ENABLED classic assets remain clawbackable through their SAC wrapper. A holder in a Soroban vault is not insulated from a Circle clawback.

### 2.5 Other (staking, perps, yield aggregators)

- **Liquid staking.** XLM is not proof-of-stake; native "staking" does not exist. **yXLM** (Ultra Capital) is a lending-receipt wrapped XLM used in Aquarius pools and Blend collateral — not a slashing-staking LST (`docs.aqua.network/technical-documents/the-aquarius-voting-mechanism`). No Ethereum-style LST exists on Stellar because the primitive is missing.
- **Perps / derivatives.** No live Soroban perp at 2026-04. CME XLM futures launched February 2026 (CeFi only; `cryptoofficiel.com/price-prediction/stellar/`). No contracts to pin — out of scope.
- **Yield aggregators beyond DeFindex.** None at mainnet scale. Blend composes with DeFindex (Yieldblox strategy). Re-evaluate if a Yearn-shaped competitor lands.
- **Soroswap aggregator, StellarBroker.** Covered in `07-dex-amm.md` §2.

## 3. Relevance to an AI-agent wallet

- **Actors served:** `A1` (rebalancing across Blend, DeFindex shares, stablecoin holdings), `A2` (orchestrator delegating budget to sub-agents paying in whichever stablecoin an invoice demands), `A3` (service-consumer paying in USDC/USDT/EURC per invoice), `A6` (read-mostly monitoring of health factor and vault report).
- **Non-negotiables:** `N1` (custody persists through bridges and lending), `N2` (no project-operated backend as single point of failure), `N5` (positions, health factors, bridge status all machine-readable), `N6` (testnet / mainnet parity — bridge testnet IDs differ per chain).
- **Threats:** `T2` (inject a borrow-and-transfer sequence), `T4` (sub-agent abuses Blend borrow authority), `T5` (USDT lookalikes, wrong bridge destination), `T6` (runaway liquidation-filling), `T7` (stale Reflector blocks repay), `T8` (bridge WASM swap under the wallet).

This is where a payment wallet becomes a DeFi wallet in fact. The decision is whether to expose lending / vault / bridge verbs as first-class tool surfaces, or treat them as contract-invocation fallbacks. §8 argues first-class: T2/T4 require typed schema enforcement at the tool boundary.

## 4. Agent-specific quirks

- **Blend's request vector is not a call.** `submit()` takes `Vec<Request>`; one transaction can atomically borrow on one reserve and repay on another. An agent trained on Aave's one-action shape under-expresses intent; the same flexibility lets a prompt-injected agent smuggle a collateral withdrawal inside a repay sequence (`pool/src/pool/actions.rs:104-118`). Render the full vector before signing.
- **Interest accrual drifts simulation → submission.** Slippage-style tolerance on "max interest paid" is not native — compute from simulation deltas.
- **Blend liquidations are auctions, not calls.** "Liquidate user X" cannot be one verb; observe an auction and fill it.
- **Stale-oracle panics are T7 + DoS.** 24-hour staleness aborts every user action including repay. Retry loops drain fees.
- **DeFindex share ratio drifts.** Share ↔ asset ratio floats because strategies accrue gains (`docs.defindex.io/api-integration-guide/troubleshooting`). Ceiling-division on withdraw can return one stroop more than requested — unit confusion under strict schema equality.
- **DeFindex upgradability is manager-held.** `upgrade(new_wasm_hash)` works unless `upgradable: false` at construction (`vault/src/interface.rs:388-400`). Third-party vaults offer *social* immutability, not on-chain. The wallet must query `upgradable`, `get_manager`, `get_emergency_manager`, `get_rebalance_manager` before recommending deposits.
- **Cross-chain is not atomic.** Axelar and Allbridge complete asynchronously. A Stellar-tx success is not a "send complete". Receipts must carry a destination-status field; a polling verb is needed.
- **Bridge fees land differently.** Axelar quotes gas in source asset via Gas Service; Allbridge deducts fee from the bridged amount. An agent sending "100 USDC" without accounting for fees lands less on destination.
- **Issuer flags are permanent.** CLAWBACK_ENABLED on USDC means Circle can burn a holder's balance (`developers.stellar.org/docs/tokens/control-asset-access`). The wallet cannot prevent this; it surfaces the flag at trustline-creation and refuses policies presuming non-clawbackable semantics.
- **USDT on Stellar is not one thing.** Multiple anchors issue "USDT". Canonicalise by issuer key, not asset code. `07-dex-amm.md` REQ-dex-token-canonical applies, escalated to stablecoin-specific policy.
- **Blend v1 vs. v2.** v1 is AGPL-3.0; v2 (`github.com/blend-capital/blend-contracts-v2`) is formally verified Q1 2025. Cross-version contract-address citation fails silently. Pin per version.
- **Axelar chain selection.** Stellar ITS runs Hub mode. An agent that accepts "any ITS-registered destination" can be pointed to a chain the user never intended.

## 5. Status on the network

- **Mainnet production.** Blend v1 (AGPL-3.0, `blend-contracts @ 895845f`); Blend v2 (formally verified Q1 2025, pools live per `defillama.com/protocol/blend-pools-v2`); DeFindex (GPL-3.0, factory + strategies deployed, 1.0.0 tagged); Axelar (Apache-2.0 — Stellar joined February 2026); Allbridge Core (audited Quarkslab March 2024); Circle USDC / EURC; FxDAO synthetics.
- **Preview / integration.** Chainlink CCIP (announced September 2025, addresses pending 2026-04); EURAU (integrated 13 April 2026). USDT0 operates on 15 chains but not Stellar at 2026-04.
- **Testnet.** All mainnet protocols have testnet deployments with different IDs; verify per network.
- **Roadmap.** Protocol 26 "Yardstick" testnet 16 April 2026, mainnet vote 6 May 2026. No protocol-layer change restructures lending or bridge primitives.
- **Breaking changes (last 12 months).** Blend v2 audit completed March 2025 (`code4rena.com/audits/2025-02-blend-v2-audit-certora-formal-verification`); USDT0 launch January 2025; Stargate DAO acquired by LayerZero August 2025 (`theblock.co/post/368040/stargate-dao-approves-layerzero-acquisition-despite-last-minute-interest-from-wormhole-axelar-and-across`); Coinbase CCIP-sole-bridge December 2025; Stellar Protocol 25 "X-Ray" mainnet 22 January 2026.

## 6. Primary documentation

All URLs accessed 2026-04-18.

- Blend — [`blend-contracts/pool/src/{contract,pool/{actions,interest,pool,health_factor}}.rs @ 895845f`](https://github.com/blend-capital/blend-contracts/blob/main/pool/src/{contract,pool/{actions,interest,pool,health_factor}}.rs @ 895845f); `docs.blend.capital/mainnet-deployments`, `docs.blend.capital/users/general-faq`; v2 at `github.com/blend-capital/blend-contracts-v2`.
- DeFindex — [`defindex/apps/contracts/vault/src/interface.rs @ f8b5c61`](https://github.com/paltalabs/defindex/blob/main/apps/contracts/vault/src/interface.rs @ f8b5c61); `docs.defindex.io/contract-deployments/mainnet-deployment`, `docs.defindex.io/api-integration-guide/troubleshooting`; SDK `github.com/paltalabs/defindex-sdk`.
- Axelar on Stellar — `github.com/axelarnetwork/axelar-amplifier-stellar`, `.../axelar-contract-deployments/blob/main/stellar/README.md`, `developers.stellar.org/docs/tools/infra-tools/cross-chain`.
- Allbridge — `github.com/allbridge-io/allbridge-core-soroban-contracts`; audit `allbridge.io/assets/docs/reports/24-01-1500-REP-Allbridge Soroban Bridge-v1.2.pdf`; `blog.quarkslab.com/allbridge-core-stellar.html`.
- Chainlink CCIP — `stellar.org/blog/foundation-news/stellar-to-join-chainlink-scale-and-adopt-data-feeds-data-streams-and-ccip-to-power-next-gen-defi-applications`.
- Stablecoins — `stellar.org/products-and-tools/circle-usdc-eurc`; `developers.circle.com/stablecoins/usdc-contract-addresses`; `theblock.co/post/380434/tether-pegged-usdt0-omnichain-stablecoin-passes-50-billion-in-cumulative-transfers`; FxDAO `fxdao.io/docs/addresses/`.
- Protocol primitives — clawback CAP-35 ([`stellar-protocol/core/cap-0035.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0035.md)); `developers.stellar.org/docs/tokens/control-asset-access`; SEP-40 ([`stellar-protocol/ecosystem/sep-0040.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0040.md)).

## 7. Known gaps or pain points

- **Axelar mainnet Soroban addresses not aggregated.** Addresses live in `axelar-chains-config` runtime JSON, not indexed on `docs.axelar.dev`. A wallet installer must fetch from the Axelar governance registry and pin the WASM hash; the project should not hardcode bridge addresses without that step.
- **CCIP on Stellar is pre-production.** Promising for T8 resistance (Chainlink DON differs from Axelar validators), but agents must not route through it yet.
- **Stablecoin lookalikes are the rule.** "USDT" collides across anchors; a non-Circle "USDC" issuer is a phishing surface. `07-dex-amm.md` REQ-dex-token-canonical is a lower bound; stablecoin handling needs its own issuer allowlist.
- **Blend oracle staleness is DoS, not just a safety feature.** 24-hour staleness aborts every user action including the repay an under-collateralised user needs. Bounded-retry policy must distinguish stale-oracle from other failures.
- **DeFindex manager trust is not self-evident.** A third-party `manager` is an omnipotent address that can pause strategies and (unless `upgradable: false`) upgrade WASM. Surfacing is the wallet's burden.
- **Bridge-to-chain mapping is policy, not default.** "Send USDC to ethereum" can route via Axelar ITS, Allbridge Core, or (when live) CCIP — different fees, finality, trust sets. No single "send" verb answers that.
- **No cross-bridge aggregator.** Unlike Soroswap aggregator for AMMs, no comparator exists at 2026-04.
- **No on-chain perp / options primitive.** Derivatives policy has no target yet; defer safely.

## 8. Candidate requirements surfaced

- `REQ-lend-submit-surface`: Wallet MUST render Blend `submit()`'s full `Vec<Request>` before signing, each request_type named (not numeric). (MUST. §4 + T2 + T3.)
- `REQ-lend-health-guard`: Wallet MUST re-simulate and refuse any Blend action whose post-condition leaves health-factor below a configured floor. (MUST. §4 + T4.)
- `REQ-lend-liquidation-verb`: Wallet MUST expose Blend liquidation as two verbs (observe auction, fill auction), never as "liquidate". (MUST. §4 + T3.)
- `REQ-lend-oracle-stale-policy`: Wallet MUST distinguish stale-oracle from other panics and back off without retry. (SHOULD. §4 + T7.)
- `REQ-lend-version-pin`: Wallet MUST pin Blend addresses per version (v1 vs v2); refuse cross-version construction. (MUST. §5 + T8.)
- `REQ-defi-vault-role-disclosure`: Before deposit to any DeFindex-shaped vault, wallet MUST surface manager, emergency-manager, rebalance-manager, and `upgradable`. (MUST. §4 + T4 + T8.)
- `REQ-defi-vault-self-managed-default`: Default policy treats vaults with non-user manager as distinct counterparty requiring explicit allowlist. (SHOULD. §3 + T5.)
- `REQ-defi-vault-min-out`: DeFindex deposit/withdraw MUST carry `amounts_min` / `min_amounts_out`; wallet computes and refuses zero. (MUST. §4 + T3.)
- `REQ-bridge-pinned-selection`: Wallet MUST require explicit bridge selection per cross-chain send; no ambient default. (MUST. §3 + T5.)
- `REQ-bridge-address-pinned-wasm`: Wallet MUST pin bridge contracts by ID + WASM hash per network; rotation requires consent. (MUST. §7 + T8.)
- `REQ-bridge-async-receipt`: Cross-chain transfers MUST be two-phase — source receipt + polling verb; "send completed" not reported on Stellar-tx success alone. (MUST. §4 + N5.)
- `REQ-bridge-fee-disclosure`: Wallet MUST display total landed amount on destination before signing. (SHOULD. §4 + T3.)
- `REQ-bridge-allowlist-counterparty`: Policy MUST support per-bridge chain and per-chain destination-address allowlists. (SHOULD. §3 + T2 + T5.)
- `REQ-stbl-issuer-pinned`: Wallet MUST canonicalise stablecoin identity by issuer key (or SEP-41 contract); policies key off canonical form. (MUST. §4 + T5.)
- `REQ-stbl-clawback-disclosure`: Wallet MUST surface CLAWBACK_ENABLED at trustline-creation and refuse to silently wrap such assets as "stablecoin". (MUST. §4 + T5.)
- `REQ-stbl-denomination-explicit`: Payment tools MUST require explicit stablecoin denomination (asset code + issuer) as typed parameter, never inferred. (MUST. §3 + T3.)
- `REQ-defi-tool-surface`: Lending, vault, bridge verbs MUST live in capability modules separate from `payment` / `swap`; policy binds each independently. (MUST. §3.)

## 9. Cross-links

- Related capability files: `02-classic-ops.md` (trustlines, clawback), `03-soroban.md` (simulation, TTL, auth entries), `04-smart-accounts.md` (policy for DeFi verbs), `05-seps.md` (SEP-40 used by Blend oracle, SEP-41 for SAC wrappers), `07-dex-amm.md` (complementary — swap venues; cite REQ-dex-token-canonical as prerequisite), `08-infra-ops.md` (bridge async polling and retry semantics).
- External candidates (under `research/external/`): none directly; closest is the Kraken-CLI / tw-agent-skills comparators for how agent-facing DeFi verbs are exposed.
- Analysis files that will cite this entry: `analysis/03-requirements.md`, `analysis/06-option-a-cli-first.md` §7, `analysis/07-option-b-agent-native.md` §7.
