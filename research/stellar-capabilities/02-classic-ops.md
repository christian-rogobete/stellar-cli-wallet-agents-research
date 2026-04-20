# Classic On-Chain Operations

**Status:** complete
**Last updated:** 2026-04-18
**ID:** 02-classic-ops

---

## 1. What it is

The non-Soroban Stellar operation set: payments, path payments, trustlines, account configuration, offers, fee bumps, sequence manipulation. Up to 100 ops per tx, gated by Low / Medium / High thresholds. Protocol-level semantics, identical across SDKs. Carries most low-risk, high-frequency agent traffic.

## 2. Components

(Primary citations in §6.)

- **Payments.** `Payment` (destination must exist; non-native assets require a trustline) and `CreateAccount`.
- **Path payments.** `PathPaymentStrictSend` (`sendAmount` + `destMin`) and `PathPaymentStrictReceive` (`sendMax` + `destAmount`) route through the classic order book and, since protocol 18, through liquidity pools as an opaque leg (CAP-24, CAP-38).
- **Trustlines.** `ChangeTrust` (limit `0` deletes). `SetTrustlineFlags` (CAP-35) supersedes `AllowTrust`; per-trustline clawback inherits the issuer's `AUTH_CLAWBACK_ENABLED_FLAG`.
- **Set options.** Master-key weight; low / med / high thresholds; signer add / update / remove (weight 0 removes); 32-byte home domain (feeds SEP-01 / SEP-10); flags `AUTH_REQUIRED=1`, `AUTH_REVOCABLE=2`, `AUTH_IMMUTABLE=4`, `AUTH_CLAWBACK_ENABLED=8`. Inflation disabled (CAP-26).
- **Thresholds.** Low: `BumpSequence`, `AllowTrust`, `SetTrustlineFlags`. Medium: `Payment`, path payments, `ChangeTrust`, offer ops, `CreateAccount`, `ManageData`, `Clawback`, non-signer `SetOptions`. High: `AccountMerge`, signer / threshold-mutating `SetOptions`. Tx authorises when summed signer weights meet the highest-threshold op.
- **Fee bumps.** `FeeBumpTransaction` wraps an already-signed V1 inner envelope; outer `feeSource` pays without consuming its sequence (CAP-15).
- **Sequences.** One slot per tx; `BumpSequence` jumps forward. CAP-21 adds `minSeqNum`, `minSeqAge`, `minSeqLedgerGap`, `ledgerBounds`, `extraSigners`.
- **Memos.** `NONE` / `TEXT (28B UTF-8)` / `ID (uint64)` / `HASH (32B)` / `RETURN (32B)`. Muxed M-accounts (CAP-27) inline a 64-bit id and are exempt from SEP-29.
- **SEP-29.** `config.memo_required=1` data entry obliges senders to attach a memo on `PAYMENT` / `PATH_PAYMENT_STRICT_*` / `MERGE_ACCOUNT` to non-muxed destinations; client-side only.
- **Classic DEX.** `ManageSellOffer`, `ManageBuyOffer`, `CreatePassiveSellOffer`. Each live offer is a subentry costing 0.5 XLM. Details in `07-dex-amm.md`.
- **Other.** `AccountMerge` (one-shot, High), `ManageData` (64/64B), `Clawback` / `ClawbackClaimableBalance` (CAP-35/36), `BumpSequence`.

## 3. Relevance to an AI-agent wallet

- **Actors served:** A1, A2, A3, A4, A5, A7.
- **Non-negotiables touched:** N1, N2, N5, N6.
- **Threats:** T1, T2, T3, T4, T5, T6, T10.

Classic ops are the primary value-bearing surface for A1 and A3, and the only on-chain delegation-scoping primitive short of Soroban: multisig thresholds + signer weights express "human + agent co-sign high, agent alone medium." Memos, SEP-29, and path-payment slippage are the fields T2 / T3 / T5 exploit most; they must be structurally explicit at the wallet boundary.

## 4. Agent-specific quirks

- **Amount-unit confusion.** SDK amounts are 7-decimal strings serialising to `int64` stroops. Raw numbers or `"1e7"` silently round. Typed tool-boundary schemas mandatory (T3).
- **Destination-must-exist.** `Payment` fails `op_no_destination` to an unfunded account; use `CreateAccount` first. Native-asset path payments auto-create; issued-asset never do.
- **SEP-29 off by default.** Stellar-core does not enforce; only the client does. Submitting direct to Horizon without the check sends to an exchange pool with no tag.
- **Memo is per-transaction.** Distinct memos to multiple destinations need muxed destinations (many anchors do not resolve them) or one tx per destination.
- **Path-payment slippage.** `destMin` / `sendMax` are the only guards. Thin-LP routes can move price between build and submit; quotes must come from fresh `strictSendPaths` / `strictReceivePaths` with a short sign window.
- **Threshold mis-set bricks the account.** `highThreshold > sum(signer weights)` with `masterKeyWeight=0` locks permanently. Simulate the post-op signer graph before signing threshold-mutating `SetOptions` (T4).
- **Fee-bump.** Inner must be V1 (CAP-15 rejects V0); some older SDK paths still emit V0. Under fee surge (100-stroop per-op floor), fee-bump rescues an already-signed tx without touching its sequence; replacement bid must be ≥ 10× the prior.
- **Sequence contention.** Two sub-agents racing on one source: loser gets `tx_bad_seq`. Mitigations: channel accounts (dedicated source per worker, fee-bumped), CAP-21 `minSeqNum`, or a serialised local sequence pool. Naïve retry-on-bad-seq double-submits when the "failed" tx was actually included.
- **Minimum-reserve floor.** `2 × base_reserve` (1 XLM at 0.5 XLM base_reserve) plus 0.5 XLM per subentry. Runaway offer / trustline creation drives the account under reserve (T6).
- **Clawback on held assets.** `AUTH_CLAWBACK_ENABLED` assets can be destroyed by the issuer at any time; not durable reserves.

## 5. Status on the network

- Mainnet + testnet: production on Protocol 25 ("X-Ray", live 2026-01-22); full parity. No breaking changes to the classic op set since protocol 19 (CAP-21, 2021) (`developers.stellar.org/docs/networks/software-versions` @ 2026-04-18).
- Roadmap: no pending CAPs reshape the classic set. Soroban work under `cap-0046-*` is orthogonal; CAP-46-06 gives a Soroban view of classic assets but does not replace any classic op.
- Known incidents: none. LaunchTube deprecation (`analysis/00-context.md` §5.4) changes submission infrastructure, not the op surface.

## 6. Primary documentation

- `docs.stellar.org/docs/learn/fundamentals/` — `transactions/list-of-operations` (ops + thresholds), `fees-resource-limits-metering`, `lumens` (reserves). All accessed 2026-04-18.
- [`stellar-protocol/core/`](https://github.com/stellar/stellar-protocol/tree/master/core) @ 44f5f5d: `cap-0015` (Fee-Bump), `cap-0021` (Preconditions), `cap-0024` (PathPayment symmetry), `cap-0027` (Muxed), `cap-0033` (Sponsored Reserve), `cap-0035` (Clawback), `cap-0038` (AMMs).
- [`stellar-protocol/ecosystem/sep-0029.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0029.md) @ 44f5f5d.
- Operator SDKs (authoritative): `stellar_flutter_sdk/` @ 21edc71, `stellar-ios-mac-sdk/` @ c24e7f387, `stellar-php-sdk/` @ d202e67, `kmp-stellar-sdk/` @ c7bfedb. Reference: `js-stellar-base/src/operations/` @ c319d2ee.

## 7. Known gaps or pain points

- **No spending-cap primitive on classic accounts.** Thresholds gate *who* signs, not *how much* per period. Caps live in policy or a smart account (`04-smart-accounts.md`).
- **No channel-account helper in SDKs.** The pattern is folklore; no operator SDK ships allocate / fund / rotate helpers.
- **SEP-29 discoverability.** Per-destination `GET /accounts/{id}` + parse `data["config.memo_required"]`; caching is safe but must be implemented.
- **Clawback not in asset metadata.** Flag on trustline + issuer account; no SEP-01 field flags it at asset level.
- **LP-only path-payment routing.** CAP-38 folded LPs into path payments opaquely; `path=[]` strict-send can route entirely through an LP with price impact unlike an order-book route.
- **Fee-bump + CAP-21 under-documented.** The ledger evaluates inner tx preconditions; bumping a tx whose `minSeqAge` has not elapsed must be read from stellar-core source.

## 8. Candidate requirements surfaced

- `REQ-classic-typed-amounts`: Amount fields typed (stroops `int64` or tagged decimal-XLM string); free-form numeric input rejected. MUST. §4, T3.
- `REQ-classic-sep29-enforce`: Wallet enforces SEP-29 on outbound `PAYMENT` / `PATH_PAYMENT_STRICT_*` / `ACCOUNT_MERGE` to non-muxed destinations; not agent-toggleable. MUST. §2, §4, T5.
- `REQ-classic-memo-explicit`: Memo type + value explicit on tool calls; no context-inferred memo. MUST. §4, T3.
- `REQ-classic-path-slippage`: Path-payment tools require `destMin` / `sendMax` from a fresh Horizon path query (configurable max age). MUST. §4, T2.
- `REQ-classic-clawback-flag`: Balance listings surface per-asset clawback status; policy may require consent before holding clawback-enabled assets above a threshold. SHOULD. §4, §7, T5.
- `REQ-classic-multisig-scoping`: `SetOptions` threshold / signer graphs express "human + agent co-sign high, agent alone medium"; wallet simulates post-op signer graph and rejects lock-out configurations. MUST. §4, T4.
- `REQ-classic-sequence-pool`: Local sequence pool (channel-account pattern) for concurrent submission under one master key; fee-bump wraps sub-transactions by default. MUST. §4, §7, A2.
- `REQ-classic-fee-payer-separable`: Fee-payer separable from signing account; fee-bump-by-default for agent-submitted tx. SHOULD. §2, A2/A4.
- `REQ-classic-reserve-guard`: Wallet refuses any op driving the source below `2 × base_reserve + 0.5 × subentries`; value refreshed from ledger. MUST. §4, T6.
- `REQ-classic-feebump-v1-check`: Fee-bump wrapping validates inner envelope type == V1. MUST. §4.
- `REQ-classic-cap21-preconds`: `minSeqNum`, `minSeqAge`, `minSeqLedgerGap`, `ledgerBounds`, `extraSigners` first-class construction fields. NICE. §7, CAP-21.
- `REQ-classic-op-preview`: Every classic op produces a human-readable preview (destination, asset, amount in stroops + XLM, memo, fee, fee-payer, network) before signing. MUST. §4, T3, T9.

## 9. Cross-links

- Related capability files: `01-accounts.md`, `03-soroban.md`, `04-smart-accounts.md`, `07-dex-amm.md`, `08-infra-ops.md`.
- External candidates: `research/external/crypto/kraken-cli.md`, `.../stellar-ecosystem/stellar-mcp-server.md`, `.../stellar-ecosystem/meridian-pay.md`.
- Analysis files: `analysis/03-requirements.md`, `06-option-a-cli-first.md` §7, `07-option-b-agent-native.md` §7.
