# DEX, AMM, and Liquidity

**Status:** in progress
**Last updated:** 2026-04-18
**ID:** 07-dex-amm

---

## 1. What it is

Stellar has a heterogeneous trading surface. Three venue families coexist on mainnet and a single trade may touch all three:

- **Classic orderbook** (SDEX) — offers via `manageSellOffer` / `manageBuyOffer` / `createPassiveSellOffer`, matched deterministically inside `stellar-core`.
- **Classic constant-product pools** — the CAP-38 AMM shape; addressed via `LiquidityPoolDeposit` / `LiquidityPoolWithdraw` and implicitly via `PathPayment*`. Fee fixed at 30 bps.
- **Soroban AMMs** — UniswapV2-shaped contracts (Soroswap, Aquarius, Phoenix) reachable only by contract invocation on SEP-41 tokens. The classic orderbook cannot be touched from Soroban; Soroban AMMs cannot be touched from classic `PathPayment*`. The two sides do not compose at the protocol level.

For an agent wallet the question is how an unattended caller expresses slippage, routing, and venue policy without producing a surprise execution, under known-hostile tool arguments. That decomposes into §2, §4, §8.

## 2. Components

URLs accessed 2026-04-18.

**Classic orderbook.** `ManageSellOfferOp` / `ManageBuyOfferOp` (offerID=0 creates, amount=0 deletes); price is a `{n,d}` rational, not a float ([`js-stellar-base/src/operations/manage_sell_offer.js:12`](https://github.com/stellar/js-stellar-base/blob/master/src/operations/manage_sell_offer.js#L12)). `CreatePassiveSellOfferOp` never takes at an equal-or-better existing price. `PathPaymentStrictSendOp` / `PathPaymentStrictReceiveOp` route through orderbook + classic pools with `destMin` / `sendMax` absolute bounds. Horizon: `/order_book`, `/paths/strict-send`, `/paths/strict-receive`, `/trade_aggregations`.

**Classic liquidity pools (CAP-38).** `LiquidityPoolDepositOp` with `maxAmountA/B` and `Price{n,d}` bounds; fails `BAD_PRICE` rather than depositing at a wrong ratio ([`stellar-protocol/core/cap-0038.md:678`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0038.md#L678)). `LiquidityPoolWithdrawOp` with `minAmountA/B`. Pool shares via `ASSET_TYPE_POOL_SHARE` trustlines, non-transferable (`cap-0038.md:50`). Fee fixed at `LIQUIDITY_POOL_FEE_V18 = 30` bps (`cap-0038.md:391`).

**Soroban AMMs.**
- **Soroswap** — UniswapV2 clone (`SoroswapFactory`, `SoroswapPair`) + `SoroswapRouter` with `swap_exact_tokens_for_tokens(amount_in, amount_out_min, path, to, deadline)`, `swap_tokens_for_exact_tokens`, `add_liquidity`, `remove_liquidity`; plus a Soroban-only aggregator across Soroswap / Phoenix / Aqua. Audits by OtterSec and Runtime Verification.
- **Aquarius** — constant-product and stable-swap pools, fees 10/30/100 bps, up to 4 tokens/pool. Mainnet entry `CBQDHNBFBZYE4MKPWBSJOPIYLW4SFSXAXUTSXJN76GNKYVYPCKWC6QUK`; `deposit`, `withdraw`, `swap`, `swap_chained`, `estimate_swap_routed`. Separate from Aquarius' older classic-pool reward layer.
- **Phoenix** — Soroban DEX hub; constant-product AMM live, orderbook and stable-swap announced.
- **Blend** — lending, not an AMM; relevant because trading policy interacts with borrow/collateral positions. Out of scope for spot swaps.
- All Soroban AMMs operate on **SEP-41 token contracts**; classic assets participate via their Stellar Asset Contract (SAC).

**Routers.** Soroswap Aggregator: on-chain, Soroban-only, excludes classic SDEX. StellarBroker: off-chain matcher combining classic SDEX, classic pools, and Soroban AMM state, submitting parallel trades — the only cross-venue router live in 2026.

**Oracles.** Reflector (SEP-40; `Pulse` free 5-min cadence, `Beam` paid with faster updates; exposes `lastprice`, `twap`, cross-price, history). DIA and Band also listed on `developers.stellar.org/docs/data/oracles/oracle-providers`.

## 3. Relevance to an AI-agent wallet

- **Actors served:** `A1` (unattended daemon rebalancing), `A3` (service-consumer paying in X from wallet holding Y), `A6` (quote-only agent).
- **Non-negotiables touched:** `N1` (self-custodial during swap), `N5` (slippage and path machine-readable), `N6` (testnet/mainnet parity — some Aquarius pool types testnet-only as of 2026-04).
- **Threats:** `T3` (hallucinated slippage, units, price ratio, path), `T5` (lookalike token contract), `T6` (repeated small swaps + MEV exhaust balance), `T7` (RPC misreports reserves or quotes).

A single `send` verb is inadequate: paying X to a counterparty expecting Y implicitly triggers a path payment with unexpressed slippage. The decision is whether the wallet exposes `swap` / `trade` as first-class verbs with explicit slippage, or delegates to a separate trading tool.

## 4. Agent-specific quirks

- **Three slippage dialects.** Classic offers use `{n,d}` rationals; path payments use `destMin` / `sendMax` absolute bounds; Soroban routers use `amount_out_min` + Unix `deadline`. Agents trained on Uniswap try to pass percents; no Stellar surface accepts one. The wallet must compute bounds from a declared tolerance and never accept an agent-supplied percent.
- **Deadline vs. preconditions.** Soroban routers need `deadline: u64`; classic ops have no per-op deadline and rely on transaction-level `timeBounds`. Both must be set — simulation succeeds without either.
- **Cross-venue routing does not compose natively.** `PathPaymentStrictSend` reaches classic orderbook + classic pools only; Soroswap aggregator reaches Soroban pools only. Crossing both requires an off-chain router (StellarBroker) or a multi-op transaction whose split the agent chooses — a direct T3/T6 surface.
- **Reserves trap on pool deposits.** Pool-share trustlines cost 0.5 XLM each; aggressive rebalancing breaches minimum reserves (`cap-0038.md:678`).
- **Token identity ambiguity.** USDC exists as `USDC:GA5Z...` (classic) and as a SEP-41 contract. Aquarius requires the contract; SDEX requires the issuer triple. Confusing the two quotes against the wrong venue. Canonicalisation lives in the wallet (T5).
- **SAC decimals mismatch.** Classic assets are 7-decimal stroops; SEP-41 tokens use i128 with their own decimals, not always 7. Unit errors here are indistinguishable from T3 hallucinations without per-call validation.
- **Soroban simulation is mandatory before signing.** Reserves move between simulation and submission; the wallet must re-simulate and abort if the refreshed quote violates declared slippage — not merely warn.
- **Oracle-to-pair mismatch.** Reflector prices use a named base asset that may not match the trade pair; cross-price needs two lookups. An agent reading "price" without confirming base and decimals compares apples to stroops.
- **MEV shape differs by venue.** `stellar-core` deterministic matching rules out mempool-level sandwiching on SDEX; Soroban transactions are reorderable within a ledger. Tight `amount_out_min` is the only defence.

## 5. Status on the network

- **Mainnet:** Classic DEX and classic pools production since Protocol 18 (November 2021). Soroswap AMM + aggregator, Aquarius (constant-product + stable-swap), Phoenix (constant-product), Reflector (`Pulse` + `Beam`), StellarBroker off-chain router — all production. Some Aquarius pool shapes still labelled testnet.
- **Testnet:** All have testnet deployments with different contract IDs. Verify per network before signing.
- **Roadmap:** CAP-82 (checked 256-bit integer arithmetic) in developer meeting notes. No active CAP restructures classic DEX fees or adds native cross-venue routing.
- **Breaking changes (last 12 months):** Aquarius Proposal #98 (20 January 2025) redirected AQUA rewards from classic pools to Aquarius Soroban AMMs. Reflector V3 audit completed October 2025 (`code4rena.com/audits/2025-10-reflector-v3`).

## 6. Primary documentation

All URLs accessed 2026-04-18.

- CAP-38 AMMs — [`stellar-protocol/core/cap-0038.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0038.md).
- SEP-40 — [`stellar-protocol/ecosystem/sep-0040.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0040.md).
- JS Base SDK — [`js-stellar-base/src/operations/{manage_sell_offer,manage_buy_offer,create_passive_sell_offer,path_payment_strict_send,liquidity_pool_deposit,liquidity_pool_withdraw}.js`](https://github.com/stellar/js-stellar-base/blob/master/src/operations/{manage_sell_offer,manage_buy_offer,create_passive_sell_offer,path_payment_strict_send,liquidity_pool_deposit,liquidity_pool_withdraw}.js).
- Horizon paths — `developers.stellar.org/docs/data/apis/horizon/api-reference/list-strict-send-payment-paths`.
- Soroswap — `docs.soroswap.finance/01-protocol-overview/01-how-soroswap-works`, `.../03-technical-reference/03-smart-contracts/04-soroswaprouter`, `docs.soroswap.finance/01-concepts/aggregator`; `github.com/soroswap/core`.
- Aquarius — `docs.aqua.network/developers/aquarius-soroban-functions`, `docs.aqua.network/aquarius-amms/pools/creating-a-pool`; `github.com/AquaToken/soroban-amm`.
- Phoenix — `phoenix-hub.io`, `github.com/Phoenix-Protocol-Group`.
- Reflector — `github.com/reflector-network/reflector-contract`, `github.com/code-423n4/2025-10-reflector`.
- StellarBroker — `github.com/stellar-broker`.
- Oracles — `developers.stellar.org/docs/data/oracles/oracle-providers`.

## 7. Known gaps or pain points

- **No protocol-level cross-venue router.** Classic SDEX and Soroban AMMs cannot compose in one operation; the client splits the trade or depends on StellarBroker. A self-custodial wallet using StellarBroker inherits a trust anchor the project otherwise avoids.
- **Slippage API heterogeneity.** No shared abstraction over `{n,d}` offers, `destMin` path bounds, and `amount_out_min` + `deadline` router calls. Every wallet builds its own — correctness-critical, no reference implementation.
- **Aggregator audit coverage uneven.** Soroswap is audited; Phoenix and Aquarius per-component audit status must be verified per release.
- **Classic pool fee fixed at 30 bps.** No tiers — agents expecting Uniswap-v3-style fee tiers are wrong on the classic side (`cap-0038.md:1407`).
- **Token identity not canonicalised.** Same asset reachable via classic issuer triple and via SEP-41 contract; no ecosystem-standard canonical form.

## 8. Candidate requirements surfaced

- `REQ-dex-slippage-explicit`: Wallet MUST require an explicit slippage bound on every trade verb; no silent default. (MUST. §4 + T3.)
- `REQ-dex-slippage-reverify`: Wallet MUST re-simulate immediately before signing and abort if the refreshed quote violates the declared bound. (MUST. §4 + T3.)
- `REQ-dex-path-explicit`: Wallet MUST surface the resolved path (venues, pools, intermediate assets) in machine-readable JSON before signing. (MUST. §3 N5 + T3.)
- `REQ-dex-venue-allowlist`: Wallet SHOULD allow policy to restrict trades to specific venues (SDEX-only, Soroswap-only, aggregator allowed). (SHOULD. §3 T5 + §7.)
- `REQ-dex-token-canonical`: Wallet MUST canonicalise token identity (classic issuer triple vs. SEP-41 contract) before routing and refuse ambiguous inputs. (MUST. §4 + T5.)
- `REQ-dex-units-typed`: Every amount in a swap tool-call MUST carry an explicit unit (stroops or SEP-41 decimals) at the schema level. (MUST. §4 + T3.)
- `REQ-dex-oracle-sanity`: Wallet SHOULD support an optional SEP-40 / Reflector sanity check that aborts a swap whose expected price deviates beyond a configured threshold. (SHOULD. §6 + T5/T7.)
- `REQ-dex-reserve-guard`: Wallet MUST refuse any deposit, offer, or trustline that would breach minimum reserves, regardless of agent instructions. (MUST. §4 + T6.)
- `REQ-dex-deadline`: Soroban-router swaps MUST set a bounded `deadline`; classic-op swaps MUST set `timeBounds`. Defaults sane; unbounded submissions rejected. (MUST. §4 + T6.)
- `REQ-dex-passive-offers-separate`: Wallet SHOULD expose `createPassiveSellOffer` as a distinct verb, not a flag, so agents cannot accidentally become takers. (SHOULD. §2 + T3.)
- `REQ-dex-third-party-router-optional`: Wallet MUST NOT depend on a non-wallet router (StellarBroker, Soroswap aggregator) for basic trading; third-party routers are opt-in. (MUST. §3 N1/N2 + §7.)
- `REQ-dex-trade-tool-surface`: Agent-facing trade tools MUST live in a dedicated capability module, not folded into `payment` / `send`, so policy rules bound them independently. (MUST. §3.)
- `REQ-dex-aggregator-pin`: If an aggregator is invoked, the wallet MUST pin its contract ID per network and refuse upgrades without user consent. (SHOULD. §7 + T8.)

## 9. Cross-links

- Related: `02-classic-ops.md` (path payments, offers, reserves), `03-soroban.md` (simulation, TTL), `04-smart-accounts.md` (policy for trade verbs), `05-seps.md` (SEP-40, SEP-41).
- External candidates: `research/external/stellar-ecosystem/freighter-walletkit.md` (xBull Swap uses Wallets-Kit); `research/external/stellar-ecosystem/meridian-pay.md` (smart-wallet shape).
- Analysis files: `analysis/03-requirements.md`, `analysis/06-option-a-cli-first.md` §7, `analysis/07-option-b-agent-native.md` §7.
