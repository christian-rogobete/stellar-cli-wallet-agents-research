# 03-requirements — stream-2 reconciliation addendum

**Addendum to** `analysis/03-requirements.md`
**Status:** 2026-04-21 — compact patch log + gap drain
**Reason for addendum-not-inline:** Two full-rewrite agent passes of the main file (~191 entries) hit tool-call size limits. This addendum records the stream-2 reconciliation as a patch log in compact form (REQ-ID + classification + actors + sources + one-line statement); the canonical full-format expansion of new entries is a follow-up task when single-file writes of that size are reliable. Downstream documents (decision matrix, recommendation) cite requirements by their final IDs as listed here.

---

## 1. Merge log

For each row, the stream-2 candidate's source is added to the existing requirement's Sources list. No statement or classification change unless noted in `## 2. Classification promotions`.

| Existing REQ-ID | Stream-2 candidate (absorbed) | Source added | Note |
|---|---|---|---|
| `REQ-acct-hardware-signer` | `REQ-acct-hardware-signer` | `research/stellar-capabilities/01-accounts.md#8` | same slug |
| `REQ-sec-policy-minimum-reserve` | `REQ-acct-reserve-floor` | `01-accounts.md#8` | widen Actors to A2; rationale sharpened re: sponsored reserves |
| `REQ-sa-c-account-support` | `REQ-acct-custom-account-native` | `01-accounts.md#8` | widen to explicit C-address handling |
| `REQ-acct-keyring-first` | `REQ-acct-key-storage-tiered` | `01-accounts.md#8` | tiered guidance (env > keyring > file) |
| `REQ-sec-network-isolation` | `REQ-acct-network-scoped-derivation` | `01-accounts.md#8` | derivation paths scoped per network |
| `REQ-sec-typed-amounts` | `REQ-classic-typed-amounts` | `research/stellar-capabilities/02-classic-ops.md#8` | stroops/XLM typed at tool boundary |
| `REQ-classic-memo` | `REQ-classic-sep29-enforce` | `02-classic-ops.md#8` | SEP-29 structural enforcement |
| `REQ-classic-memo` | `REQ-classic-memo-explicit` | `02-classic-ops.md#8` | memo type/value explicit at approval |
| `REQ-classic-sequence-pool` | `REQ-classic-sequence-pool` | `02-classic-ops.md#8`, `08-infra-ops.md#8` | **promoted SHOULD → MUST**; see §2 |
| `REQ-sec-policy-minimum-reserve` | `REQ-classic-reserve-guard` | `02-classic-ops.md#8` | reserve-arithmetic extension for Soroban footprints |
| `REQ-sor-simulate-first` | `REQ-sor-preflight-mandatory` | `research/stellar-capabilities/03-soroban.md#8` | preflight as protocol-mandatory, not optimisation |
| `REQ-sec-network-required` | `REQ-sor-network-id-pinned` | `03-soroban.md#8` | network-id in `HashIDPreimageSorobanAuthorization` |
| `REQ-obs-events-stream` | `REQ-sor-events-polling` | `03-soroban.md#8` | no native pub/sub; poll with retention awareness |
| `REQ-sa-webauthn-signer` | `REQ-sor-smart-account-webauthn` | `03-soroban.md#8` | secp256r1 via `__check_auth` |
| `REQ-sor-simulation-audit` | `REQ-sor-auth-entry-validation` | `03-soroban.md#8` | sim-auth-entries match submitted |
| `REQ-sa-webauthn-signer` | `REQ-sa-passkey-signer` | `research/stellar-capabilities/04-smart-accounts.md#8` | passkey = WebAuthn; same signer |
| `REQ-sa-session-keys` | `REQ-sa-session-rule` | `04-smart-accounts.md#8` | widen to OZ `ContextRule` shape |
| `REQ-sa-guard-hooks` | `REQ-sa-guard-equivalent` | `04-smart-accounts.md#8` | pre/post hook semantics |
| `REQ-sor-policy-contract-pinning` | `REQ-sa-attestation-optional` | `04-smart-accounts.md#8` | WASM-hash pinning of policy contracts |
| `REQ-sep-stellar-toml` | `REQ-sep-toml-fetch` | `research/stellar-capabilities/05-seps.md#8` | TTL + pinned-cert friendly |
| `REQ-acct-derivation-sep05` (split child, see §3) | `REQ-sep-key-derivation` | `05-seps.md#8` | SEP-5 / BIP-44 specifics |
| `REQ-sep-sep10-client` | `REQ-sep-10-client` | `05-seps.md#8` | structural challenge validation |
| `REQ-classic-memo` | `REQ-sep-29-enforce` | `05-seps.md#8` | third source for SEP-29 enforcement |
| `REQ-sep-sep43-walletkit` | `REQ-sep-43-adapter` | `05-seps.md#8` | SEP-43 adapter for Wallets Kit |
| `REQ-acct-recovery-optional` (new, see §4) | `REQ-sep-30-recovery-compat` | `05-seps.md#8` | SEP-30 posture |
| `REQ-sep-out-of-scope-issuance` | `REQ-sep-31-out` | `05-seps.md#8` | SEP-31 added to OUT scope |
| `REQ-sec-rpc-multi-endpoint` | `REQ-cfg-endpoints-multi` | `research/stellar-capabilities/06-tooling.md#8` | multi-endpoint with failover |
| `REQ-sec-network-required` | `REQ-cfg-network-explicit` | `06-tooling.md#8` | `--network` required on every command |
| `REQ-sec-rpc-crosscheck` (new, see §4) | `REQ-cfg-crosscheck` | `06-tooling.md#8` | source for the new entry |
| `REQ-sec-typed-amounts` | `REQ-dex-units-typed` | `research/stellar-capabilities/07-dex-amm.md#8` | asset amount + decimal typing |
| `REQ-sec-policy-minimum-reserve` | `REQ-dex-reserve-guard` | `07-dex-amm.md#8` | reserve guard on LP + DEX state |
| `REQ-classic-sequence-pool` | `REQ-perf-seq-pool` | `research/stellar-capabilities/08-infra-ops.md#8` | additional source |
| `REQ-acct-sponsored-onboarding` (new, see §4) | `REQ-perf-sponsored-reserves` | `08-infra-ops.md#8` | source for the new entry |
| `REQ-ux-idempotency-key` | `REQ-audit-submit-idempotent` | `08-infra-ops.md#8` | signed-envelope-hash as idempotency key |
| `REQ-sec-rpc-multi-endpoint` | `REQ-cfg-endpoints` | `08-infra-ops.md#8` | additional source |
| `REQ-sec-audit-log-signed` | `REQ-audit-trail` | `08-infra-ops.md#8` | signing + policy decisions logged |

**Total merges: 36.** No existing statement replaced unless noted "widen" or "sharpened"; all preserve the original REQ-ID stable.

## 2. Classification promotions

- **`REQ-classic-sequence-pool`**: SHOULD → **MUST**. Rationale: LaunchTube discontinuation + OZ Relayer Channels Plugin AGPL-3.0 clashes with N4 as a deployment dependency (see `analysis/00-context.md` §5.4). The in-process SEP-5-derived channel-account pool is therefore v1 scope, not optional. Sources: `02-classic-ops.md#8`, `08-infra-ops.md#8`, `research/external/_tier-2/stellar-ecosystem/launchtube.md` §9.

## 3. Splits

- **`REQ-sep-out-of-scope-issuance`** (existing): restated. Previously "SEP-24/SEP-6 programmatic issuance flows OUT." Now: SEP-24 **programmatic** path remains OUT; SEP-31 added to OUT (anchor-to-anchor, not client-facing; pending operator override). The **hand-off primitive** from SEP-24 splits off as new `REQ-sep-24-handoff` (SHOULD; see §4 below). SEP-6 programmatic client promoted from OUT to SHOULD as `REQ-sep-6-optional` (see §4).
- **`REQ-acct-subaccounts`** (existing): kept as parent. New child `REQ-acct-derivation-sep05` added for SEP-5 / BIP-44 specificity (see §4).

## 4. Candidates-queue resolutions

- **C10** (multi-RPC crosscheck) → promoted to **`REQ-sec-rpc-crosscheck`** (SHOULD; see §5.3 below). Remove from candidates queue in main file §0.4.
- Other candidates queue items remain as listed in the main file.

## 5. New requirements (compact form, ~85 entries)

Format per row: `REQ-ID` | `Classification` | `Actors` | one-line statement | `Sources`.

When expanded to full format in a follow-up, each entry also gets a Rationale (1-3 sentences) and an Acceptance signal.

### 5.1 Account and key management (§1.1)

- **`REQ-acct-derivation-sep05`** | MUST | A2, A7 | HD subaccount keypairs derive via SEP-5 / BIP-44 path `m/44'/148'/x'` hardened; non-hardened paths rejected. | `01-accounts.md#8`, `05-seps.md#8`, `research/brain-dump/requirements.md:23` (SPLIT child of `REQ-acct-subaccounts`)
- **`REQ-acct-muxed-outbound`** | SHOULD | A2, A3 | Wallet sends to muxed M-addresses with explicit sub-account ID rendered at approval. | `01-accounts.md#8`, T5
- **`REQ-acct-muxed-inbound`** | SHOULD | A2, A3 | Wallet accepts inbound M-addresses and presents resolved sub-account ID in receive flows. | `01-accounts.md#8`
- **`REQ-acct-sponsored-onboarding`** | SHOULD | A2 | Wallet supports CAP-0033 sponsored-reserve onboarding (Begin/End sandwich) for sub-agents without per-agent pre-funding. | `01-accounts.md#8`, `08-infra-ops.md#8`, A2
- **`REQ-acct-merge-guard`** | MUST | A5, A7 | `AccountMerge` refused if `sourceSeq >= ledgerSeq << 32` or subentries remain; error classifies the specific precondition. | `01-accounts.md#8`, T6
- **`REQ-acct-threshold-plan`** | MUST | A7 | Before signing `SetOptions` affecting thresholds or master weight, wallet simulates the post-op signer graph and refuses if it would make the account unsignable. | `01-accounts.md#8`, T1
- **`REQ-acct-subaccount-isolation`** | SHOULD | A2 | Per-sub-agent independent keypairs (non-SEP-5-derived) supported when T1 exfil risk warrants. | `01-accounts.md#8`, T1, T4
- **`REQ-acct-recovery-optional`** | NICE | A7 | SEP-30 recovery-signer attachment supported; recovery servers are user-selected, never project-operated (N3). | `01-accounts.md#8`, `05-seps.md#8`, N3

### 5.2 Classic on-chain operations (§1.2)

- **`REQ-classic-path-slippage`** | MUST | A1, A3 | Path-payment ops require explicit `destMin` (strict-send) or `sendMax` (strict-receive); no-bound paths rejected. | `02-classic-ops.md#8`, `07-dex-amm.md#8`, T3
- **`REQ-classic-clawback-flag`** | MUST | A1, A3 | Balances issued by accounts with `AUTH_CLAWBACK_ENABLED` display a named warning. | `02-classic-ops.md#8`, `09-soroban-defi.md#8`, T5
- **`REQ-classic-multisig-scoping`** | SHOULD | A2, A4 | Human+agent multisig is a first-class template (weight assignment, threshold calculator, lock-out preview). | `02-classic-ops.md#8`, A2, A4
- **`REQ-classic-fee-payer-separable`** | SHOULD | A2, A5 | Fee-payer is a distinct concept from tx source; fee-bump with different source account than signer is supported. | `02-classic-ops.md#8`, `08-infra-ops.md#8`, A5
- **`REQ-classic-feebump-v1-check`** | MUST | A5 | Fee-bump envelopes rejected if inner tx is not v1 (CAP-0015 constraint). | `02-classic-ops.md#8`, T3
- **`REQ-classic-cap21-preconds`** | SHOULD | A1 | Wallet supports CAP-21 preconditions (`minSeqNum`, `minSeqAge`, `minSeqLedgerGap`) for relaxed sequence ordering. | `02-classic-ops.md#8`, A1
- **`REQ-classic-op-preview`** | MUST | A4, A7 | Every classic-op field (destination, amount, asset, memo, path, thresholds) rendered in human-readable form at approval; raw XDR separately available for audit. | `02-classic-ops.md#8`, T2, T9

### 5.3 Soroban (§1.3)

- **`REQ-sor-auth-preimage-exact`** | MUST | A7 | Soroban auth-entry signers sign `HashIDPreimageSorobanAuthorization { networkID, nonce, signatureExpirationLedger, invocation }`, not the tx hash; test suite covers this. | `03-soroban.md#8`, T3
- **`REQ-sor-fee-headroom`** | SHOULD | A1 | Configurable headroom multiplier (default 1.3x) applied to simulation-derived `resourceFee` and resource limits before signing. | `03-soroban.md#8`, T7
- **`REQ-sor-auth-scope-display`** | MUST | A4, A7 | Soroban auth-entry invocation tree rendered hierarchically at approval (contract, function, args), not raw XDR. | `03-soroban.md#8`, T2, T9
- **`REQ-sor-nonce-expiration-strategy`** | SHOULD | A1, A3 | Auth-entry nonces chosen randomly with `signatureExpirationLedger = current + 50` by default; configurable. | `03-soroban.md#8`, T1
- **`REQ-sor-ttl-detector`** | SHOULD | A1 | Simulation `restorePreamble` output detected and presented separately from invocation cost at approval. | `03-soroban.md#8`, A1
- **`REQ-sor-ttl-maintenance`** | NICE | A1 | Scheduled `extend_footprint_ttl` job for long-running agent state under policy-bounded fee caps. | `03-soroban.md#8`, A1
- **`REQ-sor-policy-contract-pinning`** | MUST | A2 | Policy contracts called during `__check_auth` are pinned by WASM hash; upgrade raises explicit re-approval. | `03-soroban.md#8`, `04-smart-accounts.md#8`, T8
- **`REQ-sec-rpc-crosscheck`** | SHOULD | A1, A7 | For Soroban actions above a USD-value threshold, simulation cross-checked against a second independent RPC; collect signatures only on agreement. (promotion of candidate C10) | `03-soroban.md#8`, `06-tooling.md#8`, T7
- **`REQ-sor-archival-aware-ui`** | SHOULD | A1, A6 | "Entry archived" distinguished from "entry missing" in state queries; restore vs create is an explicit choice. | `03-soroban.md#8`, T3
- **`REQ-sor-auth-entry-freeze-window`** | MUST | A7 | Time between simulation and signature collection tracked; exceeding policy-configured max triggers re-simulation. | `03-soroban.md#8`, T7
- **`REQ-sor-resource-fee-display`** | SHOULD | A7 | Soroban resource fee broken out at approval into refundable (rent + events) and non-refundable (inclusion + resources). | `03-soroban.md#8`, T3
- **`REQ-sor-invoker-auth-support`** | MUST | A2 | `SorobanCredentials::SourceAccount` (invoker auth) supported alongside `SorobanCredentials::Address`. | `03-soroban.md#8`, A2

### 5.4 Smart accounts and delegation (§1.4)

- **`REQ-sa-oz-baseline`** | MUST | A2, A4 | Reference C-account uses OpenZeppelin `stellar-accounts` pinned at exact version `= 0.7.1` (not `v0.7+`; pre-1.0 library with expected breaking changes). `AuthPayload` carries `context_rule_ids: Vec<u32>` with one ID per auth context (no auto-iteration). Off-chain signing MUST compute the auth digest as `sha256(signature_payload ‖ context_rule_ids.to_xdr())` and sign that digest, not the raw `signature_payload` — this is the v0.7 hardening that closes rule-ID downgrade (T11). Version bumps are release events requiring migration review. | `04-smart-accounts.md#8`, A2, A4, T11
- **`REQ-sa-policy-selector-scope`** | NICE | A2 | Smart-account policy supports selector-level context typing (gap vs Safe modules; OZ `CallContract(Address)` is contract-scoped only, and the shipped `spending_limit` policy covers amount-level scoping but not function-selector scoping; deferred until OZ upstream or a fork ships selector-level typing). | `04-smart-accounts.md#8`, T4
- **`REQ-sa-upgrade-timelock`** | SHOULD | A7 | Smart-account upgrades via `Upgradeable` trait wrapped in wallet-enforced timelock (default 72h). `stellar-governance::timelock` is available as a separate OZ crate and can be composed in; OZ does not wire it into `stellar-accounts` by default. | `04-smart-accounts.md#8`, T8
- **`REQ-sa-fee-payer-parent`** | SHOULD | A2 | Smart-account flows can pay fees from a parent G-account via fee-bump; fee source selectable per tx. | `04-smart-accounts.md#8`, A2
- **`REQ-sa-no-relayer-hard-dep`** | MUST | A1, A2 | Smart-account flows do not depend on a specific hosted relayer; in-process submitter is the primary path. | `04-smart-accounts.md#8`, `08-infra-ops.md#8`, N2
- **`REQ-sa-delegated-signer-warn`** | MUST | A2 | Wallet warns when an agent attempts `Signer::Delegated` without explicit auth-entry assembly; recommends `Signer::External` for preflight-reliant flows. | `04-smart-accounts.md#8`, T3
- **`REQ-sa-classic-multisig-parity`** | SHOULD | A2, A4 | Classic multisig template provided with weights + thresholds + signer addition parity to smart-account session rules. | `04-smart-accounts.md#8`, `01-accounts.md#8`, A2
- **`REQ-sa-agent-usecase-example`** | SHOULD | A2, A7 | Reference C-account template + example policy configuration shipped for the A2 orchestrator use case, mirroring the OZ `stellar-accounts` README §"Use Cases / AI Agents" pattern (`Default` context type, agent key signer, whitelist + balance policies). Promoted from NICE to SHOULD given A2's centrality in the agent-wallet actor set. | `04-smart-accounts.md#8`, A7, A2
- **`REQ-sa-atomic-signer-threshold-update`** | MUST | A2, A7 | The signer-management UI bundles signer add/remove with threshold or weight updates into a single transaction via `ExecutionEntryPoint`. Signer removals that would drop the available signer count below an attached threshold are refused, or require the user to confirm an accompanying threshold decrement in the same transaction. Defends against T12 (signer-set-divergence threshold brick). | `04-smart-accounts.md#7`, OZ README §"Signer Set Divergence in Threshold Policies", T12
- **`REQ-sa-verifier-pinning`** | MUST | A2, A4, A7 | Verifier contracts (used by `Signer::External`) are pinned by wasm hash at rule-install time and re-installation is required if the wasm hash changes. The wallet refuses to install a verifier at a mutable / upgradeable deployment address unless the user explicitly opts in; the pinned wasm hash is surfaced in the approval UI. Parallel to `REQ-sor-policy-contract-pinning` for policy contracts. Defends against T13 and T14. | `04-smart-accounts.md#7`, OZ README §"Verifiers", T13, T14
- **`REQ-sa-context-rule-caps`** | SHOULD | A2, A7 | The context-rule management UI surfaces OZ's per-rule hard limits (≤15 signers, ≤5 policies) and refuses additions beyond them with an error naming the alternative (multiple rules with shared policies). Reinforces the one-rule-per-sub-agent pattern for A2 orchestrators. | `04-smart-accounts.md#2`, OZ README §"Caveats", A2
- **`REQ-sa-manager-surface`** | SHOULD | A1, A2, A5, A7 | The wallet's CLI command tree for smart-account management mirrors the manager split used by SAK and KMP `OZSmartAccountKit` (`wallet signers *`, `wallet rules *`, `wallet policies *`, `wallet credentials *`, `wallet multi-signers *`, `wallet external-signers *`). Ecosystem-familiar UX for developers already working with either SDK. | `research/external/stellar-ecosystem/smart-account-kit.md#9`, A1, A2, A5, A7
- **`REQ-sa-deterministic-address-derivation`** | SHOULD | A1, A2, A4, A5, A7 | The wallet ships a deterministic address-derivation helper using the `SHA256("openzeppelin-smart-account-kit")` seed convention shared with SAK and KMP `OZSmartAccountKit`. Every actor that needs to locate its own contract — unattended daemons (A1), orchestrators (A2), user-facing companion flows (A4), CI/CD runners with ephemeral state (A5), and the human operator (A7) — can re-derive the address from a credential + deployer without depending on a hosted indexer. Production deployments document their own branded deployer; the well-known seed is reserved for interop verification and self-discovery. | `research/external/stellar-ecosystem/smart-account-kit.md#9`, N2, N3
- **`REQ-sa-active-rule-enumeration`** | MUST | A1, A2, A4, A5, A7 | The wallet enumerates a user's active context rules without a hosted indexer dependency. Implementation options (any of): use KMP's `OZContextRuleManager.getAllContextRules()` which already does the on-chain scan natively; re-implement the same scan pattern (`get_context_rule(id)` across `[0, get_context_rules_count())` filtering deleted entries client-side) when using SAK; or maintain a local index updated on every signed rule mutation. SAK's `rules.list()` default is indexer-backed and not acceptable as the wallet's only enumeration surface. Unattended A1 and A5 actors are the sharpest case — they should not have a feature whose availability depends on a third-party endpoint being reachable. | `research/external/stellar-ecosystem/smart-account-kit.md#10`, N2, `04-smart-accounts.md#7`
- **`REQ-sa-wrapper-raw-discipline`** | SHOULD | A1, A2, A7 | The wallet wraps orchestration, session handling, signer resolution, and submission logic; exposes raw OZ contract calls (`upgrade`, `batch_add_signer`, `get_signer_id`, `get_policy_id`, `get_context_rules_count`) as an explicit escape hatch (e.g. `wallet raw <fn> [args...]`) rather than reimplementing them as first-class subcommands. Matches the SAK design principle. | `research/external/stellar-ecosystem/smart-account-kit.md#9`
- **`REQ-sa-companion-sdk-composition`** | SHOULD | A4, A7 | The companion UI's smart-account-management half is composed from SAK (TypeScript, **web-only**; demo is Vite + React with Freighter-browser-extension integration) and/or KMP `OZSmartAccountKit` (**Android, iOS, macOS, Web**; demo includes Reown / WalletConnect v2 for Freighter-Mobile on mobile) rather than built from scratch. Platform decomposition: SAK for web SPA companion surfaces if shipped; KMP for native mobile, native desktop, or unified cross-platform companion. Wallet-owned approval-channel logic (`REQ-sec-wallet-owned-approval`, `REQ-sec-approval-nonce`) is layered on top; SEP-43 and WalletConnect v2 integration inherits from the chosen SDK. Interop: both SDKs share a default deployer seed and signer wire format, so web and native surfaces are compatible and can coexist. | `research/external/stellar-ecosystem/smart-account-kit.md#9`, `REQ-ux-companion-ui`, `REQ-sep-sep43-walletkit`

### 5.5 SEP-based flows (§1.5)

- **`REQ-sep-45-client`** | SHOULD | A1, A3 | SEP-45 client (Stellar Web Auth for Contract Accounts) implemented for C-address authentication. | `05-seps.md#8`, A3
- **`REQ-sep-7-inbound`** | SHOULD | A4, A7 | SEP-7 inbound URIs accepted as untrusted input; signed artefact rendered in human-readable form before approval. | `05-seps.md#8`, T2, T9
- **`REQ-sep-41-policy`** | MUST | A1, A3 | SEP-41 `approve(spender, live_until_ledger, amount)` distinguished from `transfer` in policy; approvals gated separately with mandatory `live_until_ledger` cap. | `05-seps.md#8`, T4
- **`REQ-sep-48-typed-preview`** | SHOULD | A4, A7 | When a contract ships SEP-46 meta with SEP-48 interface spec, arg names and types rendered at approval time using the spec. | `05-seps.md#8`, T3
- **`REQ-sep-47-claim-check`** | SHOULD | A1 | SEP-47 `sep=` list queried to verify whether a contract claims to implement a standard (e.g., SEP-41) before treating it as such for policy. | `05-seps.md#8`, T3
- **`REQ-sep-53-message-sign`** | SHOULD | A3, A7 | SEP-53 sign/verify implemented for off-chain attestations; pairs with SEP-43 `signMessage`. | `05-seps.md#8`, A3
- **`REQ-sep-24-handoff`** | SHOULD | A4, A7 | Wallet authenticates via SEP-10/-45, receives interactive URL from anchor, hands off to wallet-owned companion UI (not agent-rendered). (SPLIT from `REQ-sep-out-of-scope-issuance`) | `05-seps.md#8`, A4
- **`REQ-sep-6-optional`** | SHOULD | A3 | SEP-6 programmatic client implemented; unattended flows abort on first KYC challenge unless pre-cleared. | `05-seps.md#8`, A3
- **`REQ-sep-38-quotes`** | SHOULD | A3, A4 | SEP-38 firm-quote flow: quote ID captured, quoted price displayed, signing conditional on ID-referenced terms. | `05-seps.md#8`, T3

### 5.6 DEX / AMM / liquidity (§1.7)

- **`REQ-dex-slippage-explicit`** | MUST | A1, A3 | DEX/AMM swap verbs require explicit slippage tolerance (absolute bound or percent computed to absolute); agent-supplied percent strings rejected. | `07-dex-amm.md#8`, T3
- **`REQ-dex-slippage-reverify`** | MUST | A1 | Slippage bound re-verified against current quote immediately before signing; stale quotes refreshed. | `07-dex-amm.md#8`, T7
- **`REQ-dex-path-explicit`** | SHOULD | A1 | Swap operations record explicit path (offer-book-only, pool-only, multi-hop) in the signed artefact; auto-routing gated by policy flag. | `07-dex-amm.md#8`, T3
- **`REQ-dex-venue-allowlist`** | SHOULD | A1, A3 | Policy supports per-venue allowlist (Soroswap / Aquarius / Phoenix / SDEX); out-of-allowlist routes rejected. | `07-dex-amm.md#8`, T5
- **`REQ-dex-token-canonical`** | MUST | A1, A3 | Token identity canonicalised to SEP-41 contract address (via SAC for classic assets); issuer-triples normalised before policy evaluation. | `07-dex-amm.md#8`, T5
- **`REQ-dex-oracle-sanity`** | SHOULD | A1 | For high-value swaps, execution price optionally cross-checked against Reflector oracle; abort on >N% divergence. | `07-dex-amm.md#8`, T7
- **`REQ-dex-deadline`** | MUST | A1 | Soroban AMM swap ops require a bounded Unix deadline (default current + 300s); no-deadline submissions rejected. | `07-dex-amm.md#8`, T6
- **`REQ-dex-passive-offers-separate`** | NICE | A7 | `CreatePassiveSellOffer` is a distinct verb from `ManageSellOffer`; passive offers never auto-converted. | `07-dex-amm.md#8`, A7
- **`REQ-dex-third-party-router-optional`** | NICE | A1 | StellarBroker (cross-venue router) integration is an opt-in module, not a default dependency. | `07-dex-amm.md#8`, N2
- **`REQ-dex-trade-tool-surface`** | SHOULD | A7 | Trading verbs (`swap`, `trade`, `quote`, `limit-order`) live in a dedicated `trade` subcommand / tool namespace, not mixed with `pay`. | `07-dex-amm.md#8`, A1
- **`REQ-dex-aggregator-pin`** | SHOULD | A1 | Soroswap aggregator (or equivalent) contract address + WASM hash pinned in wallet config. | `07-dex-amm.md#8`, T8

### 5.7 DeFi: lending / vault / bridge / stablecoin (§1.6 & new §1.9)

- **`REQ-lend-submit-surface`** | MUST | A1 | Blend-class lending `submit()` calls render `Vec<Request>` payload in typed form at approval (asset, amount, request type); raw-vector submission rejected. | `09-soroban-defi.md#8`, T2, T4
- **`REQ-lend-health-guard`** | SHOULD | A1 | Before signing a borrow or withdraw, wallet computes post-op health factor and refuses if below policy threshold. | `09-soroban-defi.md#8`, T6
- **`REQ-lend-liquidation-verb`** | SHOULD | A1 | Liquidation is a distinct verb with dedicated approval UX flagging liquidation-specific risk. | `09-soroban-defi.md#8`, A1
- **`REQ-lend-oracle-stale-policy`** | SHOULD | A1 | Lending operations check Reflector oracle timestamp freshness; policy defines max staleness (default 600s). | `09-soroban-defi.md#8`, T7
- **`REQ-lend-version-pin`** | MUST | A1 | Blend contract addresses + WASM hashes pinned per profile (v1 vs v2); non-pinned versions rejected. | `09-soroban-defi.md#8`, T8
- **`REQ-defi-vault-role-disclosure`** | MUST | A1, A4 | DeFindex-class vault deposits disclose current manager/admin roles at approval; non-user manager roles raise a named warning. | `09-soroban-defi.md#8`, N1, T4
- **`REQ-defi-vault-self-managed-default`** | SHOULD | A1 | Default vault template is self-managed; delegated-manager flows require explicit opt-in. | `09-soroban-defi.md#8`, N1
- **`REQ-defi-vault-min-out`** | MUST | A1 | Vault deposit / withdraw verbs require `min_out` parameter; absence is a structural error. | `09-soroban-defi.md#8`, T3
- **`REQ-defi-tool-surface`** | SHOULD | A1 | Vault verbs (`vault deposit`, `vault withdraw`, `vault rebalance`) in dedicated `vault` namespace. | `09-soroban-defi.md#8`, A1
- **`REQ-bridge-pinned-selection`** | MUST | A3 | Cross-chain bridge verbs require explicit bridge selection (Axelar / Allbridge / CCIP); no auto-routing at v1. | `09-soroban-defi.md#8`, T5
- **`REQ-bridge-address-pinned-wasm`** | MUST | A3 | Bridge contract addresses + WASM hashes pinned per bridge per profile; unpinned bridges rejected. | `09-soroban-defi.md#8`, T8
- **`REQ-bridge-async-receipt`** | SHOULD | A3 | Bridge ops return async receipt; wallet polls destination-chain finality and presents completion separately from submission. | `09-soroban-defi.md#8`, A3
- **`REQ-bridge-fee-disclosure`** | MUST | A3 | Bridge fee broken down at approval (source gas + destination gas + bridge fee + slippage); bounded by policy cap. | `09-soroban-defi.md#8`, T3
- **`REQ-bridge-allowlist-counterparty`** | SHOULD | A3 | Cross-chain destinations resolved through counterparty allowlist on destination chain; unknown requires explicit opt-in. | `09-soroban-defi.md#8`, T5
- **`REQ-stbl-issuer-pinned`** | MUST | A3 | Stablecoin issuer accounts pinned per network (USDC/Circle, EURC/Circle); "USDT" trustlines on Stellar refused by default with named T5 warning (no native Tether issuance). | `09-soroban-defi.md#8`, T5
- **`REQ-stbl-clawback-disclosure`** | MUST | A3, A7 | At trustline-creation time for a stablecoin with `AUTH_CLAWBACK_ENABLED`, named warning shown; opt-in per-trustline. | `09-soroban-defi.md#8`, T5
- **`REQ-stbl-denomination-explicit`** | MUST | A1, A3 | All stablecoin amounts denominated explicitly with asset address (SEP-41) or code+issuer; no bare-code shorthand ("USDC") without resolution. | `09-soroban-defi.md#8`, T3, T5

### 5.8 Non-functional: security / tooling / config (§2.x)

- **`REQ-cfg-endpoint-auth`** | SHOULD | A1, A7 | Per-provider auth-header schemas supported for RPC endpoints; configured per-profile, not global. | `06-tooling.md#8`, T7
- **`REQ-cfg-endpoint-pinning`** | SHOULD | A7 | TLS certificate pinning supported for user-operated endpoints; warning when an endpoint's cert changes mid-session. | `06-tooling.md#8`, T7
- **`REQ-cfg-rate-limit`** | SHOULD | A1 | HTTP 429 `Retry-After` honoured from Horizon / RPC; exponential backoff; policy-configurable max retries. | `08-infra-ops.md#8`, A1
- **`REQ-ux-friendbot-scoped`** | SHOULD | A7 | Friendbot available only on testnet/futurenet profiles; mainnet profiles reject structurally. | `06-tooling.md#8`, T10
- **`REQ-cfg-sse-reconnect`** | SHOULD | A1 | Horizon SSE consumers implement `Last-Event-ID` reconnection with backoff; no event loss within retention window. | `06-tooling.md#8`, A1
- **`REQ-cfg-retention-aware`** | SHOULD | A6 | Historical queries distinguish "no data" from "out of retention window" using RPC / Horizon metadata. | `06-tooling.md#8`, T3
- **`REQ-cfg-quickstart-support`** | NICE | A5 | Known-good profile shipped for `stellar/quickstart` local-mode testing in CI. | `06-tooling.md#8`, A5
- **`REQ-ux-explorer-optional`** | NICE | A7 | StellarExpert directory integration for counterparty identity is opt-in advisory, not normative dependency. | `06-tooling.md#8`, T5
- **`REQ-cfg-sdk-parity`** | SHOULD | A7 | For exposed features, the four operator-maintained SDKs ship parity helpers (channel pool, fee-bump wrapper, SEP-10/-45 clients). | `06-tooling.md#8`, A7
- **`REQ-ux-horizon-deprecation`** | SHOULD | A1, A6 | Data-layer uses Stellar RPC by default; Horizon calls transition-only with deprecation notice. | `06-tooling.md#8`, N6

### 5.9 Performance and ops (§2.4)

- **`REQ-perf-fee-bump`** | SHOULD | A1, A5 | Local fee-bump wrapper with policy-bounded fee caps; fee-bump source from dedicated fee-payer subaccount. | `08-infra-ops.md#8`, A1
- **`REQ-perf-soroban-smart-account-delegation`** | SHOULD | A2 | For parallel sub-agent submission, smart-account auth-entries preferred over classic sequence-number coordination; documented scale-up path. | `08-infra-ops.md#8`, A2
- **`REQ-cfg-relayer-opt-in`** | NICE | A1 | OZ Relayer (Channels Plugin) integration is explicitly opt-in; AGPL-3.0 licence implications documented at opt-in time. | `08-infra-ops.md#8`, N4
- **`REQ-perf-timebounds`** | SHOULD | A1 | Transactions built with short default `timeBounds` (current + 60s) to limit mempool replay window; configurable per flow. | `08-infra-ops.md#8`, T1

### 5.10 Machine-payment protocols — MPP (§1.6 extension)

Added 2026-04-20 based on `research/stellar-capabilities/10-mpp.md`. MPP is an HTTP-402 agentic-payment protocol from Tempo and Stripe, submitted to the IETF Datatracker as individual submission `draft-ryan-httpauth-payment` (at `-01`). Stellar support via `@stellar/mpp` (MIT, v0.5.0). Coexists with x402; both are `REQ-svc-*` peers.

- **`REQ-svc-mpp-charge`** | MUST | A1, A3 | Wallet supports MPP charge mode (all three paths: pull-sponsored, pull-unsponsored, push). Signs `SorobanCredentialsAddress` auth entries for the `transfer(from, to, amount)` host function on SAC contracts. Validates challenge parameters (amount, recipient, currency, `expires`, network) before signing. | `10-mpp.md#8`, T2, T3
- **`REQ-svc-mpp-channel`** | SHOULD | A1, A3 | Wallet supports MPP channel mode: channel open, cumulative off-chain commitment signing with a dedicated ed25519 commitment key, anti-reset discipline (persist `max(local, server-reported)` cumulative), client-initiated close. SHOULD (not MUST) because channel spec is not yet formalised. | `10-mpp.md#8`, T4
- **`REQ-svc-mpp-commitment-keys`** | SHOULD | A2 | Commitment keys are managed per channel with deterministic derivation (SEP-5-style, channel-specific path prefix) and scoped revocation; keys are not reusable across channels. | `10-mpp.md#8`, T1, T4
- **`REQ-svc-mpp-null-source-account`** | MUST | A3 | Pull-sponsored transactions correctly constructed and signed with the `GAAAAAAAAAAAAAAA...AWHF` null-source-account convention; surfaced to the user or agent as a sponsored-path indicator, not as an unknown-source warning. | `10-mpp.md#8`, T3
- **`REQ-svc-mpp-challenge-validation`** | SHOULD | A3 | 402 challenge body re-decoded and validated against the OpenAPI discovery document (if available) before signing; mismatches trigger a policy decision rather than silent acceptance. | `10-mpp.md#8`, T3, T5
- **`REQ-svc-mpp-policy-gating`** | MUST | A1, A3 | MPP charge and channel operations evaluated by the local policy engine (per-tx caps, per-period caps, counterparty allowlists, rate limits) before credential construction. The MPP spec has no human-in-the-loop gate; all authorisation discipline lives in the wallet. | `10-mpp.md#8`, T2
- **`REQ-svc-mpp-receipt-audit`** | SHOULD | A1, A7 | Every successful MPP credential-submission produces an audit record in the hash-chained log, correlating challenge ID, on-chain transaction hash (or channel commitment sequence), server receipt timestamp, and resource URL. | `10-mpp.md#8`, A7
- **`REQ-svc-mpp-x402-coexistence`** | SHOULD | A3 | When a server publishes both MPP and x402 challenges on one 402 response, the wallet chooses one protocol per request rather than dispatching both; default preference is MPP (standards-track, richer session support) with x402 as fallback. | `10-mpp.md#8`
- **`REQ-svc-mpp-version-pinning`** | SHOULD | A5, A7 | The wallet pins a specific `@stellar/mpp` SDK version and a specific spec draft version; version bumps are a wallet release event, not a silent dependency refresh. | `10-mpp.md#8`, T8
- **`REQ-svc-mpp-mcp-transport`** | SHOULD | A1, A2 | If the wallet hosts an MCP server, MPP's JSON-RPC `-32042` error and `_meta.org.paymentauth/credential` / `_meta.org.paymentauth/receipt` metadata are handled as first-class MCP protocol events. | `10-mpp.md#8`
- **`REQ-svc-mpp-channel-dispute`** | NICE | A1, A7 | When a formal channel dispute / funder-reclaim path is specified upstream, the wallet exposes it as a first-class operation (channel-force-close by funder, with time-lock). Blocked on spec availability. | `10-mpp.md#8`, T4
- **`REQ-svc-mpp-smart-account-composition`** | SHOULD | A2, A3 | When an MPP charge credential's `from` address is a C-account, the wallet constructs the auth entry's `AuthPayload` with `context_rule_ids` selected per the account's configured rules and routes signatures through the account's signer set (including `External` verifiers). MPP charge then runs under the same context-rule / signer / policy machinery as any other Soroban call on that account. | `10-mpp.md#8`, `04-smart-accounts.md#2`, T4

---

## 6. Reconciliation summary

- **Existing requirements preserved:** 108 (all IDs stable).
- **Merges applied (source-list additions):** 36.
- **Splits:** 2 (`REQ-sep-out-of-scope-issuance` restated with SEP-31 added + `REQ-sep-24-handoff` child promoted; `REQ-acct-subaccounts` kept as parent + `REQ-acct-derivation-sep05` child added).
- **Adds:** 106 new entries above (compact form): 86 from the stream-2 reconciliation 2026-04-18, plus 12 `REQ-svc-mpp-*` added 2026-04-20 for MPP coverage (including `REQ-svc-mpp-smart-account-composition`), plus 3 new `REQ-sa-*` added 2026-04-20 from the OZ `stellar-accounts` v0.7.1 review pass (`REQ-sa-atomic-signer-threshold-update`, `REQ-sa-verifier-pinning`, `REQ-sa-context-rule-caps`), plus 5 new `REQ-sa-*` added 2026-04-20 from the SAK + KMP `OZSmartAccountKit` review pass (`REQ-sa-manager-surface`, `REQ-sa-deterministic-address-derivation`, `REQ-sa-active-rule-enumeration`, `REQ-sa-wrapper-raw-discipline`, `REQ-sa-companion-sdk-composition`).
- **Classification promotions:** 2. (1) `REQ-classic-sequence-pool` SHOULD → MUST from stream-2. (2) `REQ-sa-oz-baseline` resolved to MUST (prior SHOULD classification in the addendum was an undocumented demotion from the research file's MUST); `REQ-sa-agent-usecase-example` NICE → SHOULD given A2 centrality in the agent-wallet actor set.
- **Candidates-queue resolutions:** 1 (C10 → `REQ-sec-rpc-crosscheck`).
- **Drops:** 0 — every candidate accounted for.

**Total requirement count after reconciliation: 214** (108 existing + 106 adds). Splits change phrasing of parents but not the IDs-per-count (they each add one child, already counted in the 106).

Classification split (approximate, until full-format expansion verifies): MUST ~66 (31%), SHOULD ~123 (57%), NICE ~17 (8%), OUT ~8 (4%). MUST remains under one-third (≈31%) as required.

## 7. Follow-up

The compact form above records the **vocabulary**. Full-format expansion (explicit Rationale + Acceptance signal per new entry) is deferred to a later pass when single-file writes of ~191 entries are reliable. Downstream analysis docs (`09-decision-matrix.md`, `10-recommendation.md`) cite requirements by the IDs in this addendum; the full-format expansion is a pure documentation task that does not block the decision.

Also deferred:
- Applying each merge's source-addition to the existing entries in the main `03-requirements.md` (the merge log above is the canonical record; the file edits are housekeeping).
- Updating `analysis/08-option-c-layered.md` §11 with stable REQ-IDs now that the vocabulary is locked. The entries to cite there are `REQ-sor-policy-contract-pinning`, `REQ-sec-audit-log-signed`, `REQ-ux-mcp-tool-annotations`, `REQ-ux-json-envelope`, `REQ-sep-sep43-walletkit`, `REQ-sep-48-typed-preview`, and the extensibility requirements (`REQ-ext-skill-audit-gate`, `REQ-sec-skill-signed`).

## 9. Gap drain (2026-04-21)

Provenance pointers for the `03-requirements.md` §0.3 gaps queue. Drained gaps moved to main-file §0.3.1; narrowed gaps stay in §0.3 with reduced scope.

| Gap ID | Original draft | Status | Drained by |
|---|---|---|---|
| G1 | SEP-10 ephemeral-key management | **drained** | `REQ-sep-sep10-ephemeral` (main file) |
| G3 | Minimum-reserve guard arithmetic | **narrowed** | Per-capability arithmetic absorbed into `REQ-sec-policy-minimum-reserve` via §1 merge log (stream-2 sources from `01-accounts.md`, `02-classic-ops.md`, `07-dex-amm.md`); CAP-0073 mainnet vote (2026-05-06) pending for final reserve model |
| G4 | BIP-44 derivation vs. C-account-per-child | **drained** | §3 split: `REQ-acct-derivation-sep05` + `REQ-acct-subaccount-isolation` |
| G5 | Signer-plugin interface (credential-process shape) | **drained** | `REQ-acct-signer-plugin` (interface) + `REQ-acct-hardware-signer` (Ledger) + `REQ-acct-keyring-first` (OS-keyring); passkey / TEE backends via the plugin |
| G6 | Policy-mutation audit-chain format | **drained** | `REQ-sec-audit-log-signed` (main file) — hash-chained format chosen |
| G7 | Sequence-pool behaviour under re-org / fee-bump / `txBadSeq` | **narrowed** | Fee-bump covered by `REQ-perf-fee-bump` (§4); re-org and `txBadSeq` recovery semantics remain open |

G2 (x402 payment-requirements cache) remains fully open; no REQ yet.

## 8. Date

2026-04-21. Main file date stamp (bottom of `03-requirements.md` §Appendix B) updated to 2026-04-21 in this pass.
