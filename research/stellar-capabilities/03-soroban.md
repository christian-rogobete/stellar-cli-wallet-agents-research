# Soroban

**Status:** complete
**Last updated:** 2026-04-18
**ID:** 03-soroban

---

## 1. What it is

Soroban is Stellar's smart-contract platform. For the AI-agent wallet, the relevant surface is not "how to write a contract" but **how a client assembles, authorises, simulates, pays for, and submits `InvokeHostFunctionOp`, `ExtendFootprintTTLOp`, and `RestoreFootprintOp` transactions**, and how contracts (including smart-account contracts the wallet itself may host) interact with the authorisation and state-archival machinery.

The two load-bearing facts that differ from classic Stellar:

1. **Every Soroban transaction must be preflight-simulated against a Soroban RPC node** to obtain a resource footprint, resource fee, and a set of `SorobanAuthorizationEntry` values the signer(s) must sign. A classic transaction can be built, signed, and submitted blind; a Soroban one cannot (`rs-soroban-env/soroban-simulation/src/simulation.rs @ 0068779f`).
2. **Soroban authorisation is scoped to a call tree, not to a transaction.** A signature authorises "contract A calling contract B with args X (and B calling C with args Y)"; it is independent of which account pays fees or submits. This is what makes multi-signer flows, smart accounts, and delegation possible without sharing keys (`stellar-protocol/core/cap-0046-11.md:115-155 @ 21edc71`).

## 2. Components

- **`InvokeHostFunctionOp`** — the single-op wrapper for all Soroban contract interaction; carries a `HostFunction` variant (`InvokeContract`, `CreateContract`, `UploadContractWasm`) plus an arbitrarily sized `SorobanAuthorizationEntry[]` array (`stellar-protocol/core/cap-0046-11.md:117-118 @ 21edc71`).
- **`SorobanTransactionData`** — mandatory transaction extension carrying `resources.footprint` (read-only / read-write `LedgerKey`s), `resources.instructions`, `resources.readBytes`, `resources.writeBytes`, and `resourceFee` (`stellar-protocol/core/cap-0046-07.md:194-218 @ 21edc71`).
- **`SorobanAuthorizationEntry`** with `SorobanCredentials` (`SOURCE_ACCOUNT` or `ADDRESS`) and a `SorobanAuthorizedInvocation` tree (`stellar-protocol/core/cap-0046-11.md:115-155 @ 21edc71`; `rs-soroban-sdk/soroban-sdk/src/auth.rs:62-105 @ 07757a35`).
- **`HashIDPreimageSorobanAuthorization`** — the exact byte layout the address-based signer signs over (see §3/§4).
- **`ExtendFootprintTTLOp` and `RestoreFootprintOp`** — TTL extension and archival restoration; each must be the sole op in its transaction (`stellar-protocol/core/cap-0046-12.md:483-519 @ 21edc71`).
- **Soroban RPC methods** — `simulateTransaction`, `sendTransaction`, `getTransaction`, `getEvents`, `getLedgerEntries`, `getFeeStats`, `getLatestLedger`, `getLedgers`, `getTransactions` (`stellar-rpc/cmd/stellar-rpc/internal/methods/ @ 45a1362`). No WebSocket; polling only.
- **Custom-account trait `__check_auth`** — reserved contract entry point that turns any contract into an authoriser; receives `signature_payload: Hash<32>`, an opaque `signature: Val`, and `auth_contexts: Vec<Context>` (`rs-soroban-sdk/soroban-sdk/src/auth.rs:88-104 @ 07757a35`; `stellar-protocol/core/cap-0046-11.md:377-388 @ 21edc71`).
- **Host-function crypto** — `sha256`, `keccak256`, `verify_sig_ed25519`, `recover_key_ecdsa_secp256k1`, `verify_sig_ecdsa_secp256r1`, BLS12-381, plus Poseidon/BN254 (CAP-0074/0075, protocol 25) (`rs-soroban-sdk/soroban-sdk/src/crypto.rs:120-253 @ 07757a35`).
- **OpenZeppelin `stellar-contracts` smart-account module** — context-rule-centric `__check_auth` with delegated + external signers and pluggable policy contracts (`stellar-contracts/packages/accounts/src/smart_account/mod.rs:1-100 @ 3f81125`; verifiers include ed25519 and WebAuthn/secp256r1, `stellar-contracts/packages/accounts/src/verifiers/webauthn.rs @ 3f81125`).

## 3. Relevance to an AI-agent wallet

- **Actors served:** A1, A2, A3, A4, A5, A6 (all except A7-only).
- **Non-negotiables touched:** N1 (self-custodial scoped signatures), N2 (client-side assembly, no server), N3 (preflight on user-selected RPC, not a project server), N5 (JSON-decodable simulation output), N6 (identical mechanics testnet/mainnet except fee magnitudes).
- **Threats mitigated or created:** mitigates T4 (auth entries express scoped delegation that the chain enforces), T2 (scope-and-nonce signature cannot be re-aimed by a prompt-injected agent), T10 (network id is inside the signed payload); creates pressure on T3 (wrong footprint means silent resource exhaustion at apply time), T1 (long-lived signing payload shapes — the expiration ledger and nonce are the only replay fence), and T7 (a malicious RPC can fabricate simulation results and seed footprints that do something different at apply time).

Soroban is the **substrate on which every agent-native capability rests**: smart accounts (A2/A4 delegation), x402 payment flows (A3), MPP charge mode (A3; see `10-mpp.md` §2.1, which uses a `SorobanCredentialsAddress` auth entry for SAC `transfer`), policy contracts enforcing T2/T4, and passkey-backed recovery (secp256r1 via `__check_auth`). The wallet's threat-model property that "delegation is expressed as policy, not as key sharing" (`02-threat-model.md` §6.4) is only realisable because `SorobanAuthorizationEntry` exists.

Note on MPP channel mode: MPP's channel (session) mode uses a **different** signing primitive. Rather than a `SorobanAuthorizationEntry`, the client calls `prepare_commitment(cumulativeAmount)` on a one-way-channel Soroban contract via RPC simulation, gets raw bytes back, and signs those bytes directly with a dedicated ed25519 commitment key. See `10-mpp.md` §2.2. This is the only agent-facing Soroban flow in the research that is **not** an auth-entry signature; a wallet must distinguish the two at the approval UX, because they carry different replay and scope semantics.

## 4. Agent-specific quirks

Preflight is a hard dependency. An agent cannot skip it and hand the user "the transaction" to sign; it must first round-trip to a Soroban RPC to learn the footprint, resource caps, resource fee, and — critically — the set of `SorobanAuthorizationEntry` values whose preimages the user's signer(s) must then sign. An offline wallet cannot sign a Soroban call at all without preflight output produced by some online component (`rs-soroban-env/soroban-simulation/src/simulation.rs @ 0068779f`; `stellar-rpc/cmd/stellar-rpc/internal/methods/simulate_transaction.go:157-192 @ 45a1362`).

The signer signs over `HashIDPreimageSorobanAuthorization { networkID, nonce, signatureExpirationLedger, invocation }` — SHA-256 of this preimage is what ed25519/secp256r1/custom-account signatures actually cover (`stellar-protocol/core/cap-0046-11.md:171-181 @ 21edc71`). Misunderstanding this produces a silent class of bug: signing over the transaction hash (as in classic) produces signatures that Soroban rejects.

Nonce is not a sequence number. Nonce is an arbitrary `int64` replay fence stored in a `CONTRACT_DATA` ledger entry keyed on `(address, SCV_LEDGER_KEY_NONCE)` with `durability = TEMPORARY` and `liveUntilLedgerSeq = signatureExpirationLedger`. A nonce cannot be reused until its expiration ledger passes; picking expiration too far ahead burns rent fees, picking it too close races the submission window (`stellar-protocol/core/cap-0046-11.md:470-502 @ 21edc71`). Agents that generate many short-lived auth entries must pick nonces randomly and cache expiration.

Resource declarations are **maxima that must exactly bound actual usage**, with two failure modes: (a) under-declare `instructions`/`readBytes`/`writeBytes` → apply-time `INVOKE_HOST_FUNCTION_RESOURCE_LIMIT_EXCEEDED`; (b) under-declare `resourceFee` → same class of failure even if simulation said "this is the minimum." The non-refundable part of the fee is the preflight estimate; the refundable part covers events + rent; overpayment is refunded (`stellar-protocol/core/cap-0046-07.md:224-244 @ 21edc71`). Unattended agents must add headroom to simulation output, not use it verbatim — the network can change between simulation and submission.

Footprints are exact. Reading or writing any `LedgerKey` not in the declared footprint aborts the transaction at apply time. Agents assembling transactions across RPC and submission boundaries risk the footprint going stale (e.g. a new ledger entry appears on the path) (`stellar-protocol/core/cap-0046-07.md:323-327 @ 21edc71`).

State archival can make a call that simulated clean fail at apply time. If any `PERSISTENT` entry in the transaction's footprint is archived between simulation and submission, the transaction fails with a specific `ARCHIVED` status. Soroban RPC returns a `restorePreamble` in the simulation response when this is detected (`stellar-rpc/cmd/stellar-rpc/internal/methods/simulate_transaction.go:129-154 @ 45a1362`; `stellar-protocol/core/cap-0046-12.md:228-240 @ 21edc71`). Long-running agent state (contract balances, policy rules, allowances) must either be `TEMPORARY` with a re-create-on-access pattern or `PERSISTENT` with a scheduled `extend_ttl` job the wallet can run.

Events are pulled, not pushed. Soroban RPC exposes `getEvents(startLedger, filters, pagination)`; there is no native subscription (`stellar-rpc/cmd/stellar-rpc/internal/methods/get_events.go:27-250 @ 45a1362`). Agent monitoring requires a polling loop with cursor persistence, and RPC retains events for a bounded window only. This couples unattended agent liveness to a second long-running component, not just to signing.

Policy-contract upgrade is a live threat. A contract called via `require_auth` can be upgraded by its admin using `update_current_contract_wasm` (`rs-soroban-sdk/soroban-sdk/src/deploy.rs:214-225 @ 07757a35`). Wallet policy that trusts a third-party policy contract by address silently trusts the contract's admin too. Agents delegating through third-party policy contracts need either a pinned wasm hash, a renounced admin, or a governance process.

## 5. Status on the network

- **Mainnet:** production since protocol 20; current protocol 25 ("X-Ray", live 2026-01-22). Nonce/auth-entry semantics stable since 20. State archival protocol fix (CAP-0076) applied in protocol 24 (`stellar-protocol/core/cap-0076.md:15-30 @ 21edc71`; `developers.stellar.org/docs/networks/software-versions` @ 2026-04-18).
- **Testnet:** same; futurenet tracks next protocol (26, "Yardstick") features; Protocol 26 testnet upgrade 2026-04-16, mainnet vote scheduled 2026-05-06 (`stellar.org/blog/foundation-news/stellar-yardstick-protocol-26-upgrade-guide` @ 2026-04-18).
- **Roadmap:** CAP-0071 authentication delegation for custom accounts (Draft); CAP-0072 contract signers for Stellar accounts (Draft); CAP-0074 BN254 host functions (Final, shipped in proto 25); CAP-0075 Poseidon/Poseidon2 (Final, shipped in proto 25); CAP-0077 freezable ledger keys (Accepted, proto 26); CAP-0078 limited TTL extension host functions (Accepted, proto 26); CAP-0079 muxed-address strkey host functions (Accepted, proto 26); CAP-0080 efficient ZK BN254 host functions (Accepted, proto 26); CAP-0081 TTL-ordered eviction (Accepted, proto 26); CAP-0082 checked 256-bit integer arithmetic (Accepted, proto 26) (`stellar-protocol/core/cap-007{1,2,4,5,7,8,9}.md, cap-008{0,1,2}.md @ 21edc71`; `stellar.org/blog/foundation-news/stellar-yardstick-protocol-26-upgrade-guide` @ 2026-04-18).
- **Known incidents:** the protocol-23 state-archival corruption bug affecting 478 Hot Archive entries, remediated in protocol 24 via CAP-0076 — flagged so that any wallet component ingesting historical archival state treats pre-proto-24 archived entries with caution (`stellar-protocol/core/cap-0076.md @ 21edc71`).

## 6. Primary documentation

- Soroban overview and contract-developer docs: `developers.stellar.org/docs/build/smart-contracts` (accessed 2026-04-18).
- Soroban RPC reference: `developers.stellar.org/docs/data/rpc` (accessed 2026-04-18); local: [`stellar-rpc/ @ 45a1362`](https://github.com/stellar/stellar-rpc/blob/main/ @ 45a1362).
- CAP-0046-07 (fee and resource model): [`stellar-protocol/core/cap-0046-07.md @ 21edc71`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-07.md @ 21edc71).
- CAP-0046-11 (authorisation framework): [`stellar-protocol/core/cap-0046-11.md @ 21edc71`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-11.md @ 21edc71).
- CAP-0046-12 (state archival interface): [`stellar-protocol/core/cap-0046-12.md @ 21edc71`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-12.md @ 21edc71).
- `rs-soroban-sdk` auth types: [`rs-soroban-sdk/soroban-sdk/src/auth.rs @ 07757a35`](https://github.com/stellar/rs-soroban-sdk/blob/main/soroban-sdk/src/auth.rs @ 07757a35).
- `rs-soroban-sdk` crypto host functions: [`rs-soroban-sdk/soroban-sdk/src/crypto.rs @ 07757a35`](https://github.com/stellar/rs-soroban-sdk/blob/main/soroban-sdk/src/crypto.rs @ 07757a35).
- `rs-soroban-env` simulation: [`rs-soroban-env/soroban-simulation/src/simulation.rs @ 0068779f`](https://github.com/stellar/rs-soroban-env/blob/main/soroban-simulation/src/simulation.rs @ 0068779f).
- OpenZeppelin `stellar-contracts` smart account: [`stellar-contracts/packages/accounts/src/smart_account/mod.rs @ 3f81125`](https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/mod.rs @ 3f81125).
- OpenZeppelin WebAuthn verifier (secp256r1 / passkey): [`stellar-contracts/packages/accounts/src/verifiers/webauthn.rs @ 3f81125`](https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/verifiers/webauthn.rs @ 3f81125).
- Flutter SDK auth-entry assembly reference: `stellar_flutter_sdk/lib/src/soroban/soroban_auth.dart @ 21edc71` and `lib/src/xdr/xdr_hash_id_preimage_soroban_authorization.dart`.

## 7. Known gaps or pain points

- **No native RPC subscription.** `getEvents` is poll-only; agents must maintain cursors, reconcile retention windows, and handle RPC rotation. This is the single largest operational-complexity item for A1/A2/A3 actors. See `05-infra-ops.md` (x402-MCP / LaunchTube track).
- **Simulation is not a signature-binding contract.** Between `simulateTransaction` and `sendTransaction`, the network's ledger state may change (new entry on a read path, archived entry, fee bump). The agent has no protocol-level way to guarantee "what I simulated is what will apply"; it can only narrow the race window.
- **Nonce/expiration-ledger hygiene is caller-owned.** No standard exists for how an agent picks nonces or expiration ledgers for high-volume small transactions (A3). A bad choice either burns rent fees or causes silent self-collision on retry.
- **State-archival restore costs are footprint-dependent and unbounded from the caller's point of view** — an agent that reads a long-lived allowance may discover it is ARCHIVED and must emit a `RestoreFootprintOp` (separate transaction, separate fee) before proceeding. This makes agent UX for "just send N XLM" occasionally become a two-transaction workflow.
- **Policy-contract upgrade channel has no protocol-level transparency.** An agent delegating to a third-party policy contract cannot tell from on-chain state alone that the contract may be upgraded without notice. Wallet design must surface this.
- **Third-party verifier contracts for custom accounts can panic or trap and the entire transaction fails** without a distinct error code — only `__check_auth` failure. Agent UX must surface verifier identity clearly.
- **Network identity is inside the signed payload but is supplied by the RPC** (`networkID`). A malicious RPC that returns a wrong network-passphrase hash to a naive client can produce unusable signatures; the wallet should hard-code network-passphrase-to-id for known networks. Overlaps with T7.

## 8. Candidate requirements surfaced

- `REQ-sor-preflight-mandatory`: The wallet MUST preflight every Soroban transaction via Soroban RPC and display the resulting footprint, resource fee breakdown, and authorisation-entry tree to the approval channel before signing. **MUST**. Source: §4, `02-threat-model.md` T3, T9.
- `REQ-sor-auth-preimage-exact`: The wallet MUST sign the `HashIDPreimageSorobanAuthorization` preimage (network id, nonce, expiration ledger, invocation tree) and MUST NOT sign the transaction hash alone for address-credential auth entries. **MUST**. Source: §4, CAP-0046-11.
- `REQ-sor-network-id-pinned`: The wallet MUST compute `networkID` locally from a configured network passphrase, not trust it from RPC. **MUST**. Source: §4, §7, T7, T10.
- `REQ-sor-fee-headroom`: The wallet MUST apply a configurable headroom to simulated instructions, read bytes, write bytes, and resource fee before submission; defaults visible to the user. **SHOULD**. Source: §4.
- `REQ-sor-auth-scope-display`: Approval UX MUST render the `SorobanAuthorizedInvocation` tree (contract addresses + function names + argument summary) before collecting signatures. **MUST**. Source: §3, T9.
- `REQ-sor-nonce-expiration-strategy`: The wallet MUST expose nonce selection and `signatureExpirationLedger` as explicit parameters with safe defaults (random nonce, expiration ≤ max-entry-TTL, reasonable minimum to absorb submission latency). **MUST**. Source: §4, §7.
- `REQ-sor-events-polling`: The wallet MUST provide an event-polling primitive (cursor-persistent, multi-endpoint-capable) for agent monitoring; no assumption of server-side subscriptions. **SHOULD**. Source: §2, §7.
- `REQ-sor-ttl-detector`: The wallet MUST detect and surface `restorePreamble` in simulation output and offer (or automate under policy) a `RestoreFootprintOp` + retry workflow. **SHOULD**. Source: §4, §7.
- `REQ-sor-ttl-maintenance`: For wallet-owned persistent state (smart-account rules, policy config), the wallet MUST provide a scheduled `extend_ttl` path that runs without user interaction under policy. **SHOULD**. Source: §4 long-running agent state.
- `REQ-sor-policy-contract-pinning`: Delegation to a third-party policy contract MUST record either the contract's wasm hash or an explicit "trust admin" acknowledgement; the wallet MUST warn on upgrade of an unpinned contract. **SHOULD**. Source: §4, §7, T4.
- `REQ-sor-smart-account-webauthn`: The wallet MUST support smart-account authorisation using secp256r1/WebAuthn via a custom-account verifier compatible with the OpenZeppelin `stellar-contracts` verifier interface. **SHOULD**. Source: §2, §3 A4, `01-actor-model.md`.
- `REQ-sor-auth-entry-validation`: Before signing, the wallet MUST verify that each `SorobanAuthorizationEntry` it is about to sign matches the preflight output for the transaction — no new, no substituted, no reordered entries. **MUST**. Source: §4, T3, T7.
- `REQ-sor-multi-rpc-crosscheck`: For high-value Soroban transactions, the wallet SHOULD cross-check simulation output against a second Soroban RPC endpoint before signing. **NICE**. Source: §7, T7.
- `REQ-sor-archival-aware-ui`: Human-facing surfaces MUST distinguish "submitted and succeeded," "submitted and archived-failed," and "archived-before-simulation" states; these are not interchangeable. **SHOULD**. Source: §4, §7.
- `REQ-sor-auth-entry-freeze-window`: The wallet SHOULD enforce a maximum delta between simulation-time ledger and submission-time ledger (refuse stale signatures). **SHOULD**. Source: §4.
- `REQ-sor-resource-fee-display`: The approval channel MUST separate refundable vs non-refundable resource fee components and show inclusion-fee bid separately. **SHOULD**. Source: §2, CAP-0046-07.
- `REQ-sor-invoker-auth-support`: The wallet MUST accept `SOROBAN_CREDENTIALS_SOURCE_ACCOUNT` auth entries (signed by transaction source, no extra nonce) distinct from `SOROBAN_CREDENTIALS_ADDRESS` entries, and handle both in the same approval flow. **MUST**. Source: §2, CAP-0046-11.

## 9. Cross-links

- Related capability files: `04-accounts-and-multisig.md` (classic threshold signatures that Soroban auth composes with), `06-smart-accounts-and-custom-auth.md` (`__check_auth` design space), `07-fees-and-prioritisation.md` (inclusion fee vs resource fee), `08-state-and-archival.md` (TTL, restore), `09-events-and-observability.md` (polling model).
- External candidates: `research/external/stellar-ecosystem/meridian-pay.md` (smart-account patterns), `research/external/stellar-ecosystem/freighter-walletkit.md` (client-side Soroban signing flow), `research/external/crypto/trust-wallet-agent-kit.md` and `research/external/crypto/kraken-cli.md` (agent-level abstractions over contract calls), `research/external/_tier-2/crypto/tether-wdk.md` (x402 + Soroban combined path).
- Analysis files that will cite this entry: `analysis/03-requirements.md` (all `REQ-sor-*`), `analysis/04-axes.md` (policy locus, simulation UX), `analysis/06-option-a-cli-first.md` (CLI surfaces for preflight/approve/submit), `analysis/07-option-b-agent-native.md` (typed MCP tools over the same flow).
