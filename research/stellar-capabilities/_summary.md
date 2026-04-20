# Stellar Capabilities: Cross-Cutting Summary

**Status:** stream-2 research wave complete — 2026-04-18. Addendum: MPP capability added 2026-04-20 (`10-mpp.md`).
**Depends on:** individual files `01-accounts.md` through `10-mpp.md` under `research/stellar-capabilities/`. The overview at `00-overview.md` defines scope; this file rolls up findings.

---

## Purpose

Cross-cutting rollup of the per-capability analyses. A reader can compare capability areas without opening every file. The summary is *derived* — facts live in the per-capability files. If summary and source disagree, the source wins.

## How this summary fits

Stream 2 is an inventory of **what Stellar offers that the wallet can build on** — not an evaluation of external deployment candidates (that is `research/external/_summary.md`). Candidate requirements from each file's §8 feed the candidates queue in `analysis/03-requirements.md`; cross-cutting findings inform the two option docs (`analysis/06-option-a-cli-first.md`, `analysis/07-option-b-agent-native.md`) and the decision matrix (`analysis/09-decision-matrix.md`).

---

## Per-capability matrix

| File | Area | Candidates | Key findings |
|---|---|---|---|
| `01-accounts.md` | G / C / muxed accounts, derivation, reserves, sponsorship, merge | 13 `REQ-acct-*` | `AccountMerge` sequence-number gate (`sourceSeq < ledgerSeq << 32`) is silent-failure prone; muxed-account interop is partial across counterparties despite SEP-29; C-account replay protection uses Soroban nonce + `expirationLedger`, not account sequence — agents modelling "sequence" uniformly across G and C will mis-estimate windows. |
| `02-classic-ops.md` | Payments, path payments, trustlines, multisig, fee bumps, sequence management | 12 `REQ-classic-*` (8 MUST / 2 SHOULD / 1 NICE) | Sequence-number contention across concurrent sub-agents forces a channel-account pattern (no SDK ships helpers; folklore only); **SEP-29 is client-side-only** — stellar-core does not reject memo-less payments to memo-required destinations, so the wallet is the sole enforcement point; threshold mis-set can permanently brick the account — post-op signer-graph simulation before signing `SetOptions` is mandatory. |
| `03-soroban.md` | Contract invocation, `SorobanAuthorizationEntry`, events, simulation, TTL / footprint, archival | 17 `REQ-sor-*` | **Preflight is protocol-mandatory** — a Soroban tx cannot be signed blind; the signed payload is `HashIDPreimageSorobanAuthorization` (network-ID + nonce + `signatureExpirationLedger` + invocation), **not the tx hash**; state archival makes long-running agent state second-class — `restorePreamble` detection + scheduled `extend_ttl` become wallet responsibilities. Events have no native subscription — polling only. |
| `04-smart-accounts.md` | Account abstraction: OZ `stellar-accounts`, passkey, custom policy, fee-payer, upgradeability | 12 `REQ-sa-*` | `AuthPayload` now binds `context_rule_ids` into the auth digest (v0.6 → 0.7 hardening); `Signer::Delegated` is not populated by simulation — preflight-reliant agents must use `Signer::External` with a shared verifier contract. **Explicit gap list vs. Safe modules**: module-initiated execution missing, no policy-attestation registry, no global `Guard` trait, no selector/arg-level context typing, no module-lifecycle events. Concrete backlog for our proposal. |
| `05-seps.md` | Relevance-filtered SEP index | 16 `REQ-sep-*` | 20 in-scope SEPs. New surface beyond the seed list: **SEP-46 Contract Meta**, **SEP-47 Contract Interface Discovery**, **SEP-48 Contract Interface Specification** — together they give the wallet an on-chain layer of T3 defence (typed contract invocation + human-readable preview without hand-written ABIs). **SEP-53 message signing** pairs with SEP-43. **SEP-45 confirmed** as the contract-account twin of SEP-10, using Soroban auth entries on a `web_auth_verify(...)` contract. **SEP-31 demoted to OUT** pending operator confirmation (anchor-to-anchor, not client-facing). |
| `06-tooling.md` | CLI, Horizon, Stellar RPC, SDKs, friendbot, explorers | 12 `REQ-cfg-*` / `REQ-ux-*` | **No SDF-operated mainnet Stellar RPC** — pubnet requires ecosystem providers or self-host (multi-endpoint config with per-provider auth headers is mandatory, not optional). Horizon flagged as nearing end-of-life in favour of Stellar RPC + Portfolio APIs. **StellarExpert has a public Open API** with directory endpoints (known-account categories, community-flagged fraud domains) — directly usable for the counterparty-identity layer defending T5. Stellar RPC ~7-day default retention with hard caps on `getEvents` / `getTransactions` / `getLedgers` — callers must distinguish "no data" from "out of window". |
| `07-dex-amm.md` | Classic DEX, liquidity pools, Soroban AMMs, oracles | 13 `REQ-dex-*` | **Three incompatible slippage dialects** — classic offers use `{n,d}` rational price, path payments use `destMin`/`sendMax`, Soroban routers use `amount_out_min` + Unix `deadline`; **none accept a percent**. Oracle sanity via Reflector requires cross-price + decimal + token-identity handling (classic issuer-triple vs SEP-41 contract). **Cross-venue routing does not compose at protocol level** — classic SDEX and Soroban AMMs reach each other only via StellarBroker (third-party) or a client-side split. Open question: which AMMs to support in v1 — Soroswap has strongest audit footing. |
| `08-infra-ops.md` | Sponsored reserves, fee bumps, sequence pools, LaunchTube → OZ Relayer, idempotency, audit | 10 `REQ-perf-*` / `REQ-audit-*` / `REQ-cfg-*` | **OZ Relayer Channels Plugin confirmed AGPL-3.0** — one README / package.json inconsistency flagged as vendor-side defect. Recommended v1: local SEP-5-derived channel-account pool + local fee-bump wrapper; per-agent subaccounts + smart-account policy delegation as scale-up paths. **Hash-is-idempotency-key pattern** works natively: signed envelope hash is deterministic, `DUPLICATE` on resubmit is built-in de-dup — maps onto Stripe-CLI `Idempotency-Key` semantics. **Soroban nonces are per-(address, signature)**, not per-account — smart-account signers issue parallel auth entries that classic multisig cannot match. |
| `09-soroban-defi.md` | Lending (Blend), portfolio (DeFindex), bridges, stablecoins, out-of-scope categories | 17 (5 `REQ-lend-*` + 4 `REQ-defi-*` + 5 `REQ-bridge-*` + 3 `REQ-stbl-*`) | **USDT on Stellar is a T5 trap** — no native Tether issuance as of 2026-04; any "USDT" trustline is a lookalike; wallet's counterparty-identity layer must warn or refuse by default. **USDC / EURC have `CLAWBACK_ENABLED`** — asset-hold UX must surface clawback risk. **Blend thin-wrap-of-`submit()` is wrong** — T2/T4 require typed rendering of `Vec<Request>` at the approval boundary, not a raw vector. **Axelar** (Apache-2.0) mainnet production on Stellar since Feb 2026; **Chainlink CCIP** preview only — defer routing. **Out of scope**: Wormhole, Squid, USDT0/LayerZero OFT, liquid staking, perps, yield aggregators beyond DeFindex — each with recorded reason. |
| `10-mpp.md` | Machine Payments Protocol (MPP): HTTP 402 + Stellar Soroban charge and channel modes | 11 `REQ-svc-mpp-*` | Tempo and Stripe; IETF individual submission `draft-ryan-httpauth-payment` (at `-01`). Stellar is one of several supported methods; `@stellar/mpp` SDK MIT v0.5.0 (published 2026-04-15). **Charge mode** uses `SorobanCredentialsAddress` auth entries for SAC `transfer`; three paths (pull-sponsored with null source account, pull-unsponsored, push). **Channel mode** uses a Soroban one-way payment-channel contract with off-chain cumulative commitments signed by a dedicated ed25519 commitment key separate from the account keypair; SDK implements it but no formal spec draft exists yet. **Coexists with x402** on the same 402 response (different headers). **No human-in-the-loop gate in spec** — wallet's local policy is the only defence against prompt-injected payments. **Gaps**: no channel-spec draft, no documented funder-reclaim time-lock, commitment-key management undefined, no smart-account / multisig path in the SDK. |

**Total stream-2 candidates: ~133 across content files `01`–`10`** (~122 from `01`–`09` plus 11 from `10-mpp.md`). A reconciliation pass against the existing 108 requirements in `analysis/03-requirements.md` is required; many are likely overlaps that should merge.

---

## Cross-cutting findings

### 1. Typed interfaces at the contract boundary are Stellar's T3 defence

SEP-46 (Contract Meta) + SEP-47 (Contract Interface Discovery) + SEP-48 (Contract Interface Specification) + SEP-43 (Standard Web Wallet API Interface) + SEP-53 (Sign and Verify Messages) together give an end-to-end typed surface: **contract metadata declares what interface it implements, interface-spec provides typed args including user-defined types, the wallet verifies before signing**. Paired with the MCP tool-annotation pattern from tier-2 external research, this is a genuinely strong T3/T5 architecture — on-chain typed definitions plus off-chain typed tool schemas. Our wallet should lean on all of these rather than re-invent any of them.

### 2. Concurrency for classic accounts is a first-class design problem

Sequence numbers are monotonic per G-account; parallel submissions collide as `tx_bad_seq`; multisig and sponsored reserves serialise; **no SDK ships channel-account helpers — it is folklore**. The LaunchTube deprecation and OZ Relayer's AGPL-3.0 licence force our architecture: **ship an in-process SEP-5-derived channel-account pool as a v1 MUST, not an optional fallback.** This is also a concrete opportunity to contribute helpers upstream to our maintained SDKs (Flutter, iOS, PHP, KMP).

### 3. Soroban nonces are a structural advantage we should exploit

Soroban auth nonces are per-(address, signature) with `signatureExpirationLedger`, not per-account-sequence. A smart-account signer can issue parallel auth entries with different nonces without sequence contention. **For A2 (orchestrator delegating to sub-agents), this is a first-class primitive that classic multisig cannot match.** Our smart-account architecture should be designed around this capability — session rules on a smart account let N sub-agents operate in parallel without a shared sequence lock.

### 4. Stellar's design fights ambient-state confusion — amplify it

- Network passphrase is in the transaction hash (T10 defence at the protocol level — not a convention).
- Soroban auth digest binds `context_rule_ids` (closes the rule-downgrade hole; v0.7 OZ hardening).
- Sponsored reserves require `Begin` / `End` in the same transaction.
- Muxed accounts encode sub-account ID structurally, not via memo side-channel.

The protocol's intent is explicitness. Our wallet amplifies it: require `--network` on every command, show the network tag in every response, refuse `--memo` on an M-address destination, preview the auth digest before the user signs.

### 5. Horizon is sunsetting; Stellar RPC is the forward path

Horizon is flagged as nearing end-of-life in favour of Stellar RPC + Portfolio APIs. Our wallet's data layer should prefer RPC from day one; Horizon support is for transition. **Portfolio APIs are the unknown — monitor**. This is a tracked follow-up.

### 6. No SDF mainnet RPC means multi-endpoint is mandatory

Pubnet requires ecosystem providers (Validation Cloud, Blockdaemon, Nodies) or self-hosted `stellar/quickstart`. Our wallet must support multi-endpoint config with per-provider auth-header schemas and failover. Endpoint pinning or second-RPC cross-check is required for high-value actions (T7 defence).

### 7. "No ambient simulation truth" is a standalone wallet concern

`simulateTransaction` is trusted by the wallet (footprint + auth entries + resource fees all come from it), but the RPC endpoint is in the untrusted zone (T7). A compromised RPC can fabricate simulation output. High-value actions need **endpoint pinning or a second-RPC cross-check against a diverse provider** before signature collection.

### 8. DeFi licence landscape is copyleft-heavy — clarify our posture

Blend AGPL-3.0, DeFindex GPL-3.0, OZ Relayer AGPL-3.0. **We invoke these protocols on-chain; we do not fork or embed their source.** Copyleft applies to distribution of source, not to users of deployed contracts. This distinction must be documented in the analysis record so nobody later conflates "uses Blend" with "must release wallet source under AGPL-3.0." (If we ever fork any of them, the analysis changes.)

### 9. USDT on Stellar is a phishing surface

No native Tether issuance. Any "USDT" trustline is by definition a lookalike. The wallet's counterparty-identity layer must warn or refuse transfers to "USDT" on Stellar by default, unless the operator explicitly overrides. This is MUST-ship guardrail, not a feature.

### 10. Extension-attestation patterns converge twice

Tier-1 research surfaced Safe's ERC-7484 / Rhinestone Registry and MetaMask Snaps' `(package, version, shasum)`-pinned installs. Stream-2 research surfaces OpenZeppelin's smart-account policy pattern requiring signed verifier contracts. All three converge on **signed-hash attestations keyed to the thing being trusted** (module wasm, Snap package, verifier contract). Our skills system and policy system should use one attestation mechanism across both surfaces.

### 11. HTTP-402 agentic payment is now two protocols, not one

x402 (Coinbase-origin, chain-agnostic) and MPP (Tempo and Stripe; IETF individual submission `draft-ryan-httpauth-payment` at `-01`) both extend HTTP 402 into a machine-readable payment negotiation layer. Stellar supports both: x402 documented by the SDF blog post, MPP via `@stellar/mpp` (MIT) and `draft-stellar-charge-00`. They coexist on the same 402 response (different headers) so a server can publish both and any single client can implement either. A compliant wallet treats them as peers under one `REQ-svc-*` umbrella; choosing a default is an implementation question (MPP's submission to the IETF Datatracker and its session-mode support are factors a bidder may consider). Neither provides an approval channel for human-in-the-loop gating — in both, the local policy engine is the sole defence against prompt-injection-driven legitimate payments. See `10-mpp.md` §2.5 for the coexistence matrix and the `REQ-svc-mpp-*` candidate family.

### 12. Smart-account client SDKs share a pattern — smart-account half of companion UI de-risked

Two Apache-2.0 client SDKs target OZ `stellar-accounts` v0.7.1 on-chain and share design: kalepail's TypeScript **Smart Account Kit (SAK)** for browser targets (Vite + React demo with Freighter extension) and the Kotlin-Multiplatform **`OZSmartAccountKit`** in Soneso's `kmp-stellar-sdk` for Android / iOS / macOS / Web (7-screen demo with Reown WalletConnect v2). Both expose the same manager split (`signers`, `rules`, `policies`, `credentials`, `multiSigners`, `externalSigners`, `events`) and use the same default deployer seed `SHA256("openzeppelin-smart-account-kit")` so contract addresses are interoperable from identical credential inputs (subject to byte-identical credential-ID encoding across passkey-provider outputs — open question, `smart-account-kit.md` §11). **Rule enumeration is already non-indexer-dependent in KMP** (`OZContextRuleManager.getAllContextRules()` scans the on-chain ID range and filters deleted entries client-side); **SAK's `rules.list()` defaults to the hosted indexer** and a wallet using SAK either re-implements the scan or accepts the dependency. Credential-to-contract reverse lookup defaults to the indexer in both SDKs unless the wallet uses deterministic-address derivation. **Consequence for the companion-UI story**: the smart-account-management half of the companion is a composition of SAK + KMP rather than a green-field build (`research/external/stellar-ecosystem/smart-account-kit.md`); the wallet-owned approval-channel half, SEP-43 adapter, and WalletConnect `signAuthEntry` / `signMessage` integration remain additional work. **Consequence for N2 / N3**: the indexer reads only public on-chain data, so N3 is not at risk on data-exposure grounds; the residual concerns are a third-party liveness dependency if the default URL ships unchanged, plus a query-pattern side channel (traffic analysis, not data leak). Production wallets should self-host the indexer or rely on on-chain scan + deterministic-address derivation (`REQ-sa-active-rule-enumeration`, `REQ-sa-deterministic-address-derivation`).

---

## Consolidated candidate-requirement families

Stream-2 candidates, by area-slug family:

- `REQ-acct-*` (13) — account management, derivation, reserves, sponsorship, C-account handling.
- `REQ-classic-*` (12) — classic ops: payments, trustlines, multisig, fee bumps, sequence hygiene.
- `REQ-sor-*` (17) — Soroban operations: preflight, auth-entry signing, TTL/restore, event polling.
- `REQ-sa-*` (12) — smart accounts: rule selection, auth-digest binding, delegated-signer warnings, upgradeability, fee-payer.
- `REQ-sep-*` (16) — SEP implementations: toml fetch, key derivation, SEP-10 / SEP-45 clients, SEP-29 enforcement, SEP-41 token policy, SEP-47/-48 typed preview, SEP-43 adapter.
- `REQ-cfg-*` (~10) — configuration: multi-endpoint config, endpoint auth, pinning, SSE reconnect, retention-awareness, network scoping.
- `REQ-ux-*` (~4) — CLI UX: friendbot scoping, explorer integration, Horizon-deprecation warnings.
- `REQ-dex-*` (13) — DEX/AMM: slippage unification, oracle sanity, routing correctness, venue selection.
- `REQ-perf-*` (~7) — sequence pool, fee-bump strategy, parallel-submission bounds.
- `REQ-audit-*` (~2) — signing + policy audit logging.
- `REQ-lend-*` (5) — Blend interactions: typed rendering, health-factor guard, liquidation flow, oracle staleness, version pinning.
- `REQ-defi-*` (4) — DeFindex + portfolio: vault-role disclosure, self-managed defaults, min-out guard, tool surface.
- `REQ-bridge-*` (5) — cross-chain: bridge-pinned selection, wasm pinning, async receipts, fee disclosure, counterparty allowlist.
- `REQ-stbl-*` (3) — stablecoin handling: issuer pinning, clawback disclosure, denomination explicitness.
- `REQ-svc-mpp-*` (11) — MPP: charge mode (all three paths), channel mode, commitment-key management, null-source-account handling, challenge validation, policy gating, receipt audit, x402 coexistence, version pinning, MCP transport binding, channel dispute (blocked on spec).

**Total: ~133 stream-2 candidates** to reconcile with the 108 in `analysis/03-requirements.md`.

---

## Open follow-ups

Stream-2 §7 (known gaps) items, consolidated. Each either becomes a tier-3 research task or closes through an implementation decision in the option docs / decision matrix.

1. **Pin verified mainnet Axelar Soroban addresses** (Gateway, ITS, Gas Service, Operators, Upgrader) with WASM hashes, from `axelar-chains-config`. [`09-soroban-defi.md`]
2. **Track Chainlink CCIP Stellar mainnet deployment** when addresses publish. [`09-soroban-defi.md`]
3. **Verify EURAU issuer account, flags, licence.** [`09-soroban-defi.md`]
4. **Confirm Blend v2 mainnet pool addresses and deployment timeline.** [`09-soroban-defi.md`]
5. **OZ Relayer Channels Plugin README / package.json vs LICENSE discrepancy** — file an issue with OpenZeppelin for clarification. [`08-infra-ops.md` §7, `00-context.md` §5.4]
6. **Soroban event subscription** — no native pub/sub; our polling architecture must decide cadence + retention awareness. [`03-soroban.md`]
7. **Nonce / expiration-ledger selection standard for high-frequency agents** — no community agreement; our wallet should publish a recommended policy. [`03-soroban.md`]
8. **Policy-contract-upgrade transparency** — no protocol-level signal when a called policy wasm changes; our wallet must detect and warn. [`03-soroban.md`]
9. **Simulation-to-submission race** — no protocol-level binding between simulation and submit; our wallet needs a cross-check strategy for high-value actions. [`03-soroban.md`]
10. **SEP-30 recovery** — currently draft, server-mediated model clashes with N3 unless recovery servers are user-selected. [`01-accounts.md`]
11. **Muxed-account counterparty interop survey** — partial support across exchanges / anchors; per-counterparty compatibility table needed before relying on M-accounts. [`01-accounts.md`]
12. **Horizon end-of-life schedule** — no published date or Portfolio API spec; monitor. [`06-tooling.md`]
13. **Provider-auth header schemas** — no standard across RPC vendors (Validation Cloud, Blockdaemon, Nodies all differ); our config layer must accept per-provider schema. [`06-tooling.md`]
14. **StellarExpert API machine-readable licence** — response headers do not declare a licence; assume-no-licence or reach out to StellarExpert for permission. [`06-tooling.md`]
15. **Which AMMs to support in v1** — Soroswap has strongest audit footing and aggregator; Aquarius has a prior migration incident; Phoenix is smaller. Decision input for the option docs. [`07-dex-amm.md`]
16. **Fee-bump + CAP-21 precondition edge cases** — bumping a tx whose `minSeqAge` has not elapsed is not covered by docs. [`02-classic-ops.md`]
17. **Clawback status in SEP-01 asset metadata** — no `clawback_enabled` field; wallets read the issuer flags directly. Minor spec gap. [`02-classic-ops.md`]
18. **Channel-account SDK helpers** — no SDK ships these; opportunity for our four maintained SDKs to adopt a common API. [`02-classic-ops.md`]

---

## Implications for the analysis record

- **Option A and Option B drafters both had access to most of this**, but stream-2 completed in parallel. A light review pass on `06-option-a-cli-first.md` and `07-option-b-agent-native.md` for:
  - USDT-on-Stellar trap handling (new finding).
  - OZ Relayer AGPL confirmation (both docs reference it; one paragraph update each to cite the resolution).
  - Stream-2 candidate-requirement coverage (the option docs cite REQ-IDs from the pre-stream-2 requirements doc; reconciliation will renumber some).
  - SEP-46/-47/-48 as a T3 architecture primitive both options should claim.

- **`analysis/03-requirements.md` needs a reconciliation pass**: ~122 stream-2 candidates against 108 existing requirements. Many are duplicates or near-duplicates (e.g. stream-2's `REQ-sec-*` vs. pre-stream-2's `REQ-sec-*`). Merge, renumber, and refresh source citations.

- **`analysis/08-option-c-layered.md`, `09-decision-matrix.md`, `10-recommendation.md`** still to draft. The decision matrix should score against the reconciled requirements, not the current 108.

- **Ecosystem contribution opportunities** — three surfaced in stream 2: channel-account SDK helpers in our four maintained SDKs; SEP-30 recovery-server model; a per-counterparty muxed-account interop survey. Each is a potential RFP differentiator.

---

## Cross-reference with external research

The external `_summary.md` ("consolidated what to adopt") lists ~30 patterns extracted from the 19 external candidates. Where stream-2 findings confirm or extend those:

- **Chain-namespaced tool names + CAIP-2** (external, Phantom MCP §9) + **Soroban network passphrase in tx hash** (stream-2 §3 `03-soroban.md`) = aligned T10 defence across transport and protocol.
- **Wallet-issued nonce on commit step** (external, Phantom MCP §10) + **Soroban auth-digest binds `context_rule_ids`** (stream-2 `04-smart-accounts.md` §4) = aligned T2 / T4 defence; smart-account nonces are already a wallet-issued artefact at the protocol level.
- **Attestation registry for extensions** (external, Safe §9 + Snaps §9) + **OZ `stellar-accounts` verifier-contract model** (stream-2 `04-smart-accounts.md` §2) = signed-hash attestations are the convergent pattern across skills, policies, and on-chain verifiers.
- **In-process sequence pool as first-class** (external, LaunchTube tier-2 §9) + **SEP-5 channel-account pool** (stream-2 `08-infra-ops.md` §9) = same design, now with an explicit Stellar-native implementation path.

These alignments make the option docs' architecture claims stronger, because the external patterns we propose to adopt are realisable with Stellar-native primitives we just documented.
