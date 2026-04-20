# Accounts and Keys

**Status:** complete
**Last updated:** 2026-04-18
**ID:** 01-accounts

---

## 1. What it is

Stellar exposes three address shapes that a wallet must handle. **G-accounts** are the classic on-ledger account object keyed by an Ed25519 public key; they own balances, hold signers, carry flags, consume a minimum XLM balance per subentry, and advance by sequence number. **C-accounts** are Soroban contracts used as first-class authorisers via the Soroban Authorization Framework; any contract implementing the reserved `__check_auth` function becomes a custom account for auth purposes ([`stellar-protocol/core/cap-0046-11.md:340-388`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-11.md:340-388) @ `44f5f5d`). **M-accounts** (muxed) are a strkey-level wrapper around a G-account plus a 64-bit subaccount ID; they do not exist on the ledger but appear in transaction and operation source/destination fields ([`stellar-protocol/core/cap-0027.md:47-78`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0027.md:47-78) @ `44f5f5d`).

Keys are Ed25519. SEP-05 specifies BIP-39 mnemonics with SLIP-0010 derivation along `m/44'/148'/x'` (coin type 148, hardened account index); `m/44'/148'/0'` is the primary ([`stellar-protocol/ecosystem/sep-0005.md:34-55`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0005.md:34-55) @ `44f5f5d`). Strkey encodes key and address types with distinct prefixes (G, S, M, C, T, X, P, L, B) ([`stellar-protocol/ecosystem/sep-0023.md:45-56`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0023.md:45-56) @ `44f5f5d`). The operator-maintained SDKs all implement SEP-05 (Flutter `lib/src/sep/0005/wallet.dart` @ `21edc71`, iOS `stellarsdk/libs/HDWallet/Mnemonic.swift` @ `c24e7f3`, PHP `Soneso/StellarSDK/SEP/Derivation/Mnemonic.php` @ `d202e67`, KMP `stellar-sdk/.../sep/sep05/Mnemonic.kt` @ `c7bfedb`).

## 2. Components

- **G-account object.** Account ID (Ed25519), sequence number, balances, signers, thresholds, flags, per-subentry reserve accounting ([docs.stellar.org accounts page](https://developers.stellar.org/docs/learn/fundamentals/stellar-data-structures/accounts) accessed 2026-04-18).
- **Signers and thresholds.** Master + up to 20 extra signers per transaction; weights 0-255; operations classified low/medium/high and require cumulative weight to meet the relevant threshold (docs.stellar.org signatures-multisig, accessed 2026-04-18). Signer types: ED25519 public key, PreAuthTx, HashX (`sep-0023.md:45-56`), and signed-payload signers introduced in CAP-0040 ([`stellar-protocol/core/cap-0040.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0040.md) @ `44f5f5d`).
- **Account flags.** `AUTH_REQUIRED`, `AUTH_REVOCABLE`, `AUTH_IMMUTABLE`, `AUTH_CLAWBACK_ENABLED`; authorisation-state transitions are constrained by `AUTH_REVOCABLE_FLAG` ([`stellar-protocol/core/cap-0018.md:85`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0018.md#L85) @ `44f5f5d`).
- **Base reserve & subentries.** One base reserve = 0.5 XLM; each trustline, offer, extra signer, and data entry consumes one base reserve; hard cap 1,000 subentries per account (docs.stellar.org accounts page, accessed 2026-04-18).
- **Sponsored reserves.** `BeginSponsoringFutureReservesOp` / `EndSponsoringFutureReservesOp` (must appear in the same transaction), `RevokeSponsorshipOp`; `AccountEntryExtensionV2` tracks `numSponsored`/`numSponsoring` and per-signer sponsor IDs ([`stellar-protocol/core/cap-0033.md:47-88`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0033.md:47-88) @ `44f5f5d`).
- **AccountMerge operation.** One-shot deletion that transfers all XLM to a destination; preconditions include no trustlines/offers/data/extra signers, no active sponsorships, no `AUTH_IMMUTABLE`, and `sourceSequence < ledgerSeq << 32` (docs.stellar.org list-of-operations accountMerge, accessed 2026-04-18).
- **Muxed accounts (M-).** `MuxedAccount` XDR union with an optional 64-bit `id`; supported as operation source, fee source, payment destination, accountMerge destination, clawback target ([`stellar-protocol/core/cap-0027.md:60-79`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0027.md:60-79) @ `44f5f5d`; docs.stellar.org pooled-accounts-muxed-accounts-memos, accessed 2026-04-18).
- **SEP-29.** Memo-required flag via the `config.memo_required` data entry; senders must check before payment/path-payment/accountMerge *when the destination is not muxed* ([`stellar-protocol/ecosystem/sep-0029.md:42-60`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0029.md:42-60) @ `44f5f5d`).
- **C-accounts and `__check_auth`.** Soroban custom account trait `CustomAccountInterface::__check_auth(env, signature_payload, signatures, auth_contexts)`; receives the full `auth_contexts: Vec<Context>` tree via pre-order DFS of `rootInvocation` ([`rs-soroban-sdk/soroban-sdk/src/auth.rs:88-104`](https://github.com/stellar/rs-soroban-sdk/blob/main/soroban-sdk/src/auth.rs:88-104) @ `07757a3`; [`stellar-protocol/core/cap-0046-11.md:368-445`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-11.md:368-445) @ `44f5f5d`). Self-reentrancy for `__check_auth` is the only permitted contract self-reentry (`cap-0046-11.md:446-452`).
- **C-account lifecycle.** Contracts are created from a `Wasm` executable uploaded via `HOST_FUNCTION_TYPE_UPLOAD_CONTRACT_WASM` and instantiated with `HOST_FUNCTION_TYPE_CREATE_CONTRACT` under a deterministic `CONTRACT_ID_PREIMAGE_FROM_ADDRESS` (address + salt) or `_FROM_ASSET` preimage ([`stellar-protocol/core/cap-0046-02.md:62-100`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-02.md:62-100) @ `44f5f5d`). Upgradeability is contract-internal (typically via a host-function call on a stored admin address).
- **Derivation paths.** SEP-05 (BIP-39 + SLIP-0010, `m/44'/148'/x'`). Only hardened child derivation is valid for Ed25519 (`sep-0005.md:57-69` @ `44f5f5d`).

## 3. Relevance to an AI-agent wallet

- **Actors served.** A1 (automation daemon; sequence and reserve hygiene), A2 (orchestrator; SEP-05 subaccount derivation *and* C-account delegation), A3 (service-consumer agent; muxed destinations for anchors/exchanges), A4 (user-facing assistant; flag-driven auth holds), A5 (CI/CD deploy agent; AccountMerge as teardown), A6 (read-mostly; derivation without on-chain funding), A7 (human operator; recovery via SEP-30).
- **Non-negotiables touched.** N1 (self-custodial: G/C/M keys never leave the host without consent), N2/N3 (no central server: derivation and signer management are local), N6 (testnet/mainnet parity: strkey and SEP-05 identical both sides; only flag defaults differ).
- **Threats mitigated or created.** T1 (key exfiltration: mnemonic + SLIP-0010 concentrates risk unless subaccount keys are isolated), T4 (delegated-authority abuse: C-accounts with `__check_auth` are the only mechanism that expresses scoped delegation on-chain without key sharing), T5 (counterparty impersonation: muxed destinations interact with SEP-29 memo-required logic), T6 (runaway loop: minimum-reserve floor must be protected by wallet policy), T10 (network confusion: muxed-address handling must include network passphrase context because strkeys are network-agnostic).

The capability area is where delegation lives. The CLI-vs-agent-native decision (`analysis/06`, `analysis/07`) reduces in large part to *where A2's per-subagent bounds are expressed*: a single mnemonic with SEP-05 subaccounts leans CLI-first and off-chain-policy; per-sub-agent C-accounts with `__check_auth` policy lean agent-native and on-chain.

## 4. Agent-specific quirks

- **SEP-05 derivation is one-way for signing, but every subaccount is a separate G-account that must be funded and reserve-metered.** Creating 20 sub-agents means 20 × 0.5 XLM base + one per-signer/trustline/data entry. An unattended orchestrator (A2) that mints subaccounts on demand must either pre-fund or use sponsored reserves (CAP-0033) with `Begin/End` pairs in the same transaction. The sponsorship lifecycle is *not* set-and-forget: `RevokeSponsorshipOp` transfers sponsorship to the owner, who must then have the reserve available.
- **AccountMerge is one-shot and the sequence-number gate is exotic.** Source sequence must be `< ledgerSeq << 32`. An A5 CI/CD agent that bumps sequence aggressively on an ephemeral key may hit this and not realise, since the error surfaces as a merge failure rather than a sequence overflow. Merging also invisibly requires that every subentry be removed first — a common failure mode when an agent forgets a trustline.
- **Muxed-account interop is partial and silent.** Per CAP-0027 + the muxed-accounts guide, M-addresses are valid in payment destination and operation source fields but rejected in many *account* fields, producing runtime errors. Exchanges and anchors still widely require memos (SEP-29 exempts muxed destinations, but counterparty systems may not). An agent sending to an exchange must detect and prefer memo form; the wallet should resolve the canonical form per counterparty before signing.
- **C-account auth is replay-protected by Soroban nonce, not account sequence.** A C-account signer never increments the G-account sequence; `SOROBAN_CREDENTIALS_ADDRESS` carries its own `nonce` and `expirationLedger`, bounded by `stateArchivalSettings.maxEntryTTL - 1` (`cap-0046-11.md:454-468`). Agents that model "sequence" uniformly across G/C will mis-estimate signature expiry and replay windows.
- **Muxed strkey round-tripping from contracts landed in protocol 26.** `strkey_to_muxed_address` / `muxed_address_to_strkey` are Accepted for protocol 26 ([`stellar-protocol/core/cap-0079.md:1-45`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0079.md:1-45) @ `44f5f5d`). Contracts that want to consume muxed destinations (e.g. a router paying an exchange) could not do this cleanly before protocol 26.
- **Hardware-wallet support is wallet-layer, not SDK-layer.** None of the operator-maintained SDKs (Flutter/iOS/PHP/KMP) embeds Ledger integration; hardware signing is the caller's responsibility. Freighter and Stellar Wallets Kit supply this at the UX layer.

## 5. Status on the network

- **Mainnet.** G-accounts, sponsorship, and signed-payload signers: production (final). Muxed accounts: production since protocol 13; counterparty interop incomplete. Soroban custom accounts and `__check_auth`: production since protocol 20 (`cap-0046-11.md` @ `44f5f5d`). CAP-0073 (SAC can create G-account trustlines/balances) and CAP-0079 (muxed strkey host functions): Accepted, protocol 26 (`cap-0073.md:1-30`, `cap-0079.md:1-11` @ `44f5f5d`).
- **Testnet.** Matches mainnet. Protocol 26 features reach testnet before mainnet by convention; verify the active protocol via `getNetwork` on Soroban RPC before relying on CAP-0073/0079/0077.
- **Roadmap.** CAP-0077 (freeze ledger keys via network config) and CAP-0078 (limited TTL extension host functions) are Accepted for protocol 26. CAP-0073 enables in-Soroban G-account creation — reshapes the onboarding flow for sub-agents (A2) because the orchestrator can fund and create sub-agent G-accounts from a Soroban call atomically.
- **Breaking changes in the last 12 months.** None specific to account shape or signer types; CAPs 67/73/77/78/79 extend behaviour without breaking existing encodings.

## 6. Primary documentation

- CAP-0027 muxed accounts ([`stellar-protocol/core/cap-0027.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0027.md) @ `44f5f5d`; <https://github.com/stellar/stellar-protocol/blob/master/core/cap-0027.md>), accessed 2026-04-18.
- CAP-0033 sponsored reserves ([`stellar-protocol/core/cap-0033.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0033.md) @ `44f5f5d`), accessed 2026-04-18.
- CAP-0040 signed-payload signers ([`stellar-protocol/core/cap-0040.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0040.md) @ `44f5f5d`), accessed 2026-04-18.
- CAP-0046-02 smart contract lifecycle ([`stellar-protocol/core/cap-0046-02.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-02.md) @ `44f5f5d`), accessed 2026-04-18.
- CAP-0046-11 Soroban authorization framework ([`stellar-protocol/core/cap-0046-11.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-11.md) @ `44f5f5d`), accessed 2026-04-18.
- CAP-0073 SAC G-account create/trust ([`stellar-protocol/core/cap-0073.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0073.md) @ `44f5f5d`), accessed 2026-04-18.
- CAP-0079 muxed strkey host functions ([`stellar-protocol/core/cap-0079.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0079.md) @ `44f5f5d`), accessed 2026-04-18.
- SEP-05 key derivation ([`stellar-protocol/ecosystem/sep-0005.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0005.md) @ `44f5f5d`), accessed 2026-04-18.
- SEP-23 strkeys ([`stellar-protocol/ecosystem/sep-0023.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0023.md) @ `44f5f5d`), accessed 2026-04-18.
- SEP-29 account memo requirements ([`stellar-protocol/ecosystem/sep-0029.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0029.md) @ `44f5f5d`), accessed 2026-04-18.
- SEP-30 account recovery ([`stellar-protocol/ecosystem/sep-0030.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0030.md) @ `44f5f5d`), accessed 2026-04-18.
- `rs-soroban-sdk` custom-account interface ([`rs-soroban-sdk/soroban-sdk/src/auth.rs`](https://github.com/stellar/rs-soroban-sdk/blob/main/soroban-sdk/src/auth.rs) @ `07757a3`), accessed 2026-04-18.
- SDK SEP-05 implementations: Flutter `lib/src/sep/0005/wallet.dart` @ `21edc71`; iOS `stellarsdk/libs/HDWallet/Mnemonic.swift` @ `c24e7f3`; PHP `Soneso/StellarSDK/SEP/Derivation/Mnemonic.php` @ `d202e67`; KMP `stellar-sdk/src/commonMain/kotlin/com/soneso/stellar/sdk/sep/sep05/Mnemonic.kt` @ `c7bfedb`.
- docs.stellar.org *Accounts* <https://developers.stellar.org/docs/learn/fundamentals/stellar-data-structures/accounts>, accessed 2026-04-18.
- docs.stellar.org *Signatures and Multisig* <https://developers.stellar.org/docs/learn/fundamentals/transactions/signatures-multisig>, accessed 2026-04-18.
- docs.stellar.org *Pooled Accounts, Muxed Accounts, Memos* <https://developers.stellar.org/docs/build/guides/transactions/pooled-accounts-muxed-accounts-memos>, accessed 2026-04-18.

## 7. Known gaps or pain points

- **One mnemonic as single point of total loss.** SEP-05 puts every derived subaccount key under one seed; compromising the seed compromises every A2 sub-agent. Mitigations (passphrase, split storage, per-sub-agent independent keypairs) are all the wallet's responsibility, not SEP-05's.
- **Sponsorship lifecycle is transactional, not session-scoped.** There is no "maintain sponsorship" primitive: every new sponsored entry needs a `Begin/End` pair in a transaction. Long-running orchestrators (A2) must batch subaccount creation aggressively to avoid per-child fee overhead.
- **Counterparty M-account support is uneven.** Exchanges and anchors historically require memos even when SEP-29 permits muxed destinations. Agents cannot assume M-account form is accepted; the wallet should maintain a per-counterparty preference cache.
- **No in-protocol key-rotation operation for G-accounts.** Rotation is `SetOptions` to add a new signer, raise thresholds, then demote or remove the old master. Agents that try to "rotate" without planning for threshold math will lock themselves out.
- **SEP-30 recovery is draft-status and server-mediated.** The only standardised recovery path depends on third-party recovery servers, which clashes with N3 (no central server) unless the user chooses to trust specific servers explicitly.
- **`__check_auth` has no standardised policy language.** Every custom account invents its own signature schema and policy vocabulary. OpenZeppelin `stellar-contracts` and kalepail's `smart-account-kit` are converging references (see `04-smart-accounts.md`) but not canonical.

## 8. Candidate requirements surfaced

- `REQ-acct-derivation-sep05`: wallet MUST generate and re-derive keys along SEP-05 `m/44'/148'/x'` for interop with ecosystem wallets. Proposed MUST. Source: `research/stellar-capabilities/01-accounts.md#2`.
- `REQ-acct-hardware-signer`: wallet SHOULD support Ledger (and other hardware) signing for A7 on mainnet actions above a policy threshold. Proposed SHOULD. Source: `#3, #4`.
- `REQ-acct-muxed-outbound`: wallet MUST produce and parse M-addresses for outbound payments, and MUST fall back to memo form when the counterparty signals SEP-29 memo-required. Proposed MUST. Source: `#4, SEP-29`.
- `REQ-acct-muxed-inbound`: wallet SHOULD advertise a canonical inbound address form per use case (M- for agent-internal multiplexing, G-+memo when integrating with legacy counterparties). Proposed SHOULD. Source: `#4`.
- `REQ-acct-sponsored-onboarding`: wallet MUST support sponsored-reserve onboarding (CAP-0033 `Begin/End` pairs) as the default funding flow for A2 sub-agent subaccounts. Proposed MUST. Source: `#4`.
- `REQ-acct-reserve-floor`: wallet MUST refuse any operation that would take the source below its computed minimum balance, regardless of agent instruction. Proposed MUST. Source: `#2, T6`.
- `REQ-acct-merge-guard`: wallet MUST refuse AccountMerge when subentries, sponsorships, or `AUTH_IMMUTABLE` would cause failure, and MUST surface the specific blocker. Proposed MUST. Source: `#2, docs.stellar.org`.
- `REQ-acct-custom-account-native`: wallet MUST treat C-accounts as first-class signers, including producing `SOROBAN_CREDENTIALS_ADDRESS` entries with proper nonce and `expirationLedger`. Proposed MUST. Source: `#2, CAP-0046-11`.
- `REQ-acct-threshold-plan`: wallet SHOULD execute signer/threshold changes as atomic multi-operation transactions with pre-flight threshold verification. Proposed SHOULD. Source: `#7`.
- `REQ-acct-key-storage-tiered`: wallet MUST store root seed in OS keyring / TPM / hardware-backed storage by default and document the backend chosen per platform. Proposed MUST. Source: `#3, T1`.
- `REQ-acct-subaccount-isolation`: wallet SHOULD support per-sub-agent independent keypairs (not all SEP-05 children) as an option for A2 orchestrators to bound T1 blast radius. Proposed SHOULD. Source: `#7, T1`.
- `REQ-acct-network-scoped-derivation`: wallet MUST namespace and display derived addresses per network passphrase to prevent T10 collisions. Proposed MUST. Source: `#4, T10`.
- `REQ-acct-recovery-optional`: wallet MAY expose SEP-30 recovery with an explicit, per-user selection of recovery servers; MUST NOT default to project-operated servers. Proposed NICE. Source: `#7, N3`.

## 9. Cross-links

- Related capability files: `02-classic-ops.md` (SetOptions, ChangeTrust, sequence-number mechanics), `03-soroban.md` (`SorobanAuthorizationEntry`, nonce, `expirationLedger`), `04-smart-accounts.md` (reference C-account implementations and policy patterns), `05-seps.md` (SEP-05, SEP-23, SEP-29, SEP-30), `08-infra-ops.md` (sequence pools, fee-bump composition).
- External candidates (under `research/external/`) that interact: `stellar-ecosystem/meridian-pay.md`, `stellar-ecosystem/stellar-mcp-server.md`, `stellar-ecosystem/freighter-walletkit.md`, `_tier-2/crypto/phantom-mcp-server.md`, `_tier-2/crypto/tether-wdk.md`.
- Analysis files that will cite this entry: `analysis/03-requirements.md` (candidate queue), `analysis/06-option-a-cli-first.md` and `analysis/07-option-b-agent-native.md` §7 (subaccount vs smart-account delegation model, including C-account delegation via `__check_auth`).
