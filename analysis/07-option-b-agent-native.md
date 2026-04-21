# Option B: agent-protocol-first with a CLI surface

**Status:** draft — 2026-04-21
**Audience:** RFP Track delegates
**Depends on:** `00-context.md`, `01-actor-model.md`, `02-threat-model.md`, `03-requirements.md`, `research/external/_summary.md`

---

## 1. Definition

Option B treats a **normative tool-call protocol definition** as the canonical artefact of the Stellar AI-Agent Wallet. The wallet is, first, a signer service described by a strictly-typed schema: JSON Schema for every tool's inputs and outputs, CAIP-2 network identifiers on every call (`stellar:pubnet`, `stellar:testnet`), explicit Stellar-specific signing semantics (classic envelopes, `SorobanAuthorizationEntry`, SEP-10 challenge, SEP-43 WalletConnect messages), and versioned conformance profiles that callers can negotiate and test against. The canonical form of "what the wallet can do" is a spec document plus machine-readable schemas, not a binary's `--help` output.

Three transports are generated from that schema and share one dispatch:

1. An **MCP server over stdio**, the primary surface for agent callers (`REQ-ux-mcp-stdio`). Every feature is an MCP tool first.
2. A **CLI** (`stellar-ai ...`), a projection of the same schema with table/JSON output and human affordances (`REQ-ux-json-default`, `REQ-ux-table-render`, `REQ-ux-setup-wizard`). The CLI exists because A7 (human operator) is a first-class actor and must never be worse off than the agent (`01-actor-model.md` §3 A7).
3. A **library** (Rust crate with language bindings generated from the same schema) for embedded callers (`REQ-ext-library-crate`).

A WalletConnect v2 endpoint on `stellar:pubnet` / `stellar:testnet` and a SEP-43 interface on the companion UI are *additional* derived surfaces — not parallel designs — so dapps and mobile approvers see the same typed operations as agents (`REQ-sep-sep43-walletkit`, `REQ-ux-companion-ui`).

This option takes one upfront architectural bet: specify the schema precisely, then treat every transport as a projection. The MoonPay Open Wallet Standard (`research/external/crypto/moonpay-agents-ows.md` §9) is the closest real-world analog, though OWS has no Stellar chain profile; the Stellar profile would need to be defined as part of this option's scope. The option's commitment is that the schema is the product, and every binary is a conformance-tested projection of it.

## 2. Centre of gravity and what that implies

The centre of gravity is the **schema**, not the MCP server. This distinction matters because it is the part most likely to be misread.

"Agent-protocol-first" does **not** mean "the MCP server is the only surface." It means: (i) the wallet's behaviour is defined by a versioned spec and a set of JSON Schemas before any transport is built; (ii) adding a feature means editing the spec first and then generating bindings for MCP, CLI, library, WalletConnect, and companion UI from a single source of truth; (iii) conformance is checked against the spec, not against a reference binary; (iv) a third party could re-implement the wallet in another language by reading the spec, and the canonical test suite would validate their implementation against the reference.

What that implies:

- **Every transport is derived, including the CLI.** CLI argument parsing, help text, exit codes, and JSON envelope shape come from the schema. The CLI is hand-polished at the presentation layer but not at the operation-definition layer. This is the cost: the CLI cannot grow features the schema does not already name, and extending it is not just a Rust change.
- **Conformance is testable.** Any reimplementation can be run against the same tool-call fixtures and produce byte-identical receipts. This is the property that makes the schema a real artefact rather than decoration.
- **Version negotiation is explicit.** The spec carries a version; callers declare which version they speak; the server responds with the intersection. Breaking changes are spec-level events with migration notes, not silent CLI regressions.
- **MCP annotations are first-class.** `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` (`REQ-ux-mcp-tool-annotations`) are spec fields. The MCP server projects them; the CLI uses them to decide interactive prompts; the library exposes them for embedding callers.

The centre of gravity is what is **canonical**. The transports an operator reaches for is a separate question answered per-actor in §5.

## 3. Concrete architecture

### 3.1 The tool-call protocol: canonical schema and versioning

The protocol is a family of documents under `spec/`, modelled on OWS's seven-sub-spec discipline (`research/external/crypto/moonpay-agents-ows.md` §9 item 1): `00-core` (envelope, errors, version negotiation); `01-identity-and-keys` (keyring, external-signer, hardware profile; `REQ-acct-signer-plugin`, `REQ-acct-hardware-signer`); `02-signing-interface` (Stellar-specific: `stellar_sign_classic_envelope`, `stellar_sign_auth_entry` per `REQ-sor-auth-entries`, `stellar_sign_sep10_challenge`, `stellar_sign_message`; deliberately not `sign(bytes)` because policy cannot reason about opaque bytes — C9, `research/external/_tier-2/crypto/privy.md` §10); `03-policy-engine`; `04-agent-access-layer` (MCP stdio, CLI, library, WalletConnect — each preserves the same dispatch, envelope, audit side-effects); `05-chain-profile-stellar` (CAIP-2, G/C/M address encodings, asset encoding, SEP-10/SEP-43 bindings, Soroban auth-entry shape); `06-conformance` (golden fixtures; mandatory profile declarations — no blanket "compliance").

Schemas are JSON Schema 2020-12 with Stellar-specific typed refinements (`StellarG`, `StellarC`, `StellarAsset`, `StellarMemo`), analogous to Phantom MCP's `EthereumAddressSchema` / `SolanaAddressSchema` (`research/external/_tier-2/crypto/phantom-mcp-server.md` §9). Amounts are dual-unit `{ amountUnit: "ui"|"base", decimals, value }` (`REQ-sec-typed-amounts`). Parse errors are typed: `EMPTY_STRING`, `INVALID_FORMAT`, `EXCESSIVE_PRECISION` (`REQ-sec-typed-amount-errors`, `research/external/_tier-2/crypto/tether-wdk.md` §9 item 4).

Versioning is semver on the spec; tools carry `sinceVersion`; server advertises `protocolVersions`; the intersection governs the session. This is where wallet-issued nonces are baked in from day one (strategic finding 5 in `research/external/_summary.md`).

### 3.2 Primary transports: MCP stdio, CLI, library

**MCP over stdio** is the primary agent transport, launched by the same binary (`stellar-ai mcp`, `REQ-ux-mcp-stdio`), reusing the CLI dispatch path: Kraken's pattern reinterpreted with MCP canonical rather than derived (`research/external/crypto/kraken-cli.md` §9). Tool names are chain-namespaced (`stellar_*`, `REQ-ux-mcp-tool-names-namespaced`) with CAIP-2 per schema. Annotations are non-optional (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`; `REQ-ux-mcp-tool-annotations`). Read/write subsets are separately selectable (`REQ-ux-mcp-read-write-groups`, Tether WDK pattern). A6 can instantiate a server that structurally cannot sign.

**CLI** is a derived surface with three output modes — JSON envelope (`REQ-ux-json-default`, `REQ-ux-json-envelope`), table (`REQ-ux-table-render`), `--jq`/`--query` projection (`REQ-ux-query-projection`) — standardised exit codes mapping to spec error categories (`REQ-ux-exit-codes`, `REQ-ux-error-taxonomy`), and dangerous-tool acknowledgement gate (Kraken's `acknowledged=true` pattern; `REQ-sec-dangerous-tool-gate`). Honest constraint: the CLI feels less hand-crafted than a CLI-first design, because its surface is a schema projection. Human operators will sometimes ask for flags the schema does not name; the answer is to edit the spec.

**Library** ships as a Rust crate (`REQ-ext-library-crate`) with TS/Python bindings from the same schema. A library caller hits the same dispatch as MCP and CLI: identical audit log, policy engine, receipt shape.

**WalletConnect v2** (`stellar:pubnet`/`stellar:testnet`) and **SEP-43** are additional derived surfaces (`REQ-sep-sep43-walletkit`). SEP-43 shape is verbatim from `research/external/stellar-ecosystem/freighter-walletkit.md` §9, extended with `signAuthEntry` and `signMessage`. These are the methods the incumbent Creit WC module rejects.

### 3.3 Policy engine as a normatively-specified component

Policy is a spec-defined component with a JSON rule grammar. Rule types in `spec/03-policy-engine.md`: `amountCap` (USD-cents and native, Coinbase CDP `changeCents` pattern, `research/external/crypto/coinbase-agentic.md` §9; `REQ-sec-policy-per-tx-cap`); `periodCap` (`REQ-sec-policy-per-period-cap`); `counterpartyAllowlist` with Fireblocks-style typed object kinds (`INTERNAL_WALLET` / `EXTERNAL_ADDRESS` / `CONTRACT` / `SEP10_IDENTITY` / `HOME_DOMAIN` / `ASSET_ISSUER`) mapping to Stellar G/C-account and SEP-10/home-domain distinctions (`research/external/_tier-2/crypto/fireblocks.md` §9); `rateLimit`; `minimumReserveGuard` (arithmetic open in G3); `sorobanResourceCap` (`REQ-sor-footprint-cap`); `executablePlugin` with 5-second timeout and fail-closed semantics (OWS pattern, `research/external/crypto/moonpay-agents-ows.md` §9 item 3).

This option explicitly rejects OWS's choice of keeping caps and allowlists out of the declarative layer (`research/external/crypto/moonpay-agents-ows.md` §10 item 1). Per-tx, per-period, and counterparty allowlists are first-class declarative rules.

Evaluation is AND across rules, first-match-stops. Every denial is logged with the draft it refused (`REQ-audit-policy-denials`). Policy files are owner-signed (`REQ-sec-policy-signed`, Privy `owner_id` pattern); mutations require owner signature (`REQ-sec-policy-mutation-governance`). Because policy is in the spec, a compliant reimplementation evaluates the same rules on the same fixtures.

### 3.4 Smart-account delegation

Delegation is expressed as Soroban smart-account (C-account) contracts with programmable `__check_auth` (`REQ-sa-c-account-support`), built on OpenZeppelin `stellar-accounts` v0.7.1 **context rules plus policies** — signers and policies bound to specific `Context` types with optional `valid_until`, the caller explicitly selecting which rule governs each auth context (`research/stellar-capabilities/04-smart-accounts.md` §3). **OZ context rules are the delegation primitive**, not Safe-style module attachment — OZ does not expose modules (`research/external/crypto/safe.md` §9 discusses the Safe-module pattern as a comparator, not a reference). Module-attestation registry (Safe/Rhinestone / ERC-7484 shape; C1) is a v1.x extension; the spec reserves the namespace.

The reference account template combines: **recurring-payment and top-up flows as custom wallet machinery** on top of OZ context rules (`REQ-sa-policy-initiated-exec`, reframed: OZ `Policy::enforce()` can only accept or revert inside `__check_auth` — it cannot originate transactions, so the wallet ships a scheduler or keeper that signs the next invocation under a dedicated rule whose policy caps amount, counterparty, and period); **a WebAuthn passkey signer** extending Meridian Pay's ~100-LOC baseline (`research/external/stellar-ecosystem/meridian-pay.md` §9 item 2) but inspecting `_auth_contexts` — which Meridian Pay does not (`research/external/stellar-ecosystem/meridian-pay.md` §10 item 1); **expiring session keys via on-ledger `valid_until`** on the rule (Turnkey-equivalent pattern expressed natively in OZ); **revocation enforced at the next `__check_auth`** (`REQ-sa-revocation-takes-effect-before-next-sig`; forward-looking, not retroactive — already-signed envelopes in flight can still succeed if submission races the revocation ledger); and **root-quorum break-glass** via a dedicated `Default`-context rule for A7 recovery (C4). Delegation is never key sharing (`REQ-sa-delegation-as-policy-not-key`).

Signer-set hygiene is schema-enforced: **atomic signer-and-threshold updates** via `ExecutionEntryPoint` (`REQ-sa-atomic-signer-threshold-update`) and **verifier pinning by wasm hash** at rule-install time (`REQ-sa-verifier-pinning`) are MUST-level requirements in the spec, not wallet-optional conventions. The canonical multi-agent pattern is **one context rule per sub-agent** rather than many signers on a shared rule — OZ's 15-signer per-rule hard cap makes per-sub-agent rules the correct architecture, matches the OZ README §"Use Cases / AI Agents" template, and preserves the isolation property that revoking one sub-agent affects only that sub-agent's rule.

### 3.5 Wallet-issued commit nonce and approval channel

The two-step simulate-then-commit pattern is from Phantom MCP (`research/external/_tier-2/crypto/phantom-mcp-server.md` §9 item 1), but fixed as its §10 item 2 calls for: `confirmed` is **not** an agent-passed boolean. It is a wallet-issued, wallet-signed nonce returned by simulate, cryptographically bound to payload hash and identity (`REQ-svc-x402-nonce-binding`, `REQ-sec-approval-nonce`).

Sequence: (1) agent calls `stellar_*_simulate` → wallet returns `{ simulation, commitNonce }`; (2) agent or a human (via companion UI) calls `stellar_*_commit` with `{ commitNonce }`; (3) wallet verifies the nonce is its own, matches the payload hash, is within TTL, and has not been replayed. Because the spec mandates the binding, every transport enforces it.

The **approval channel is wallet-owned** (`REQ-sec-wallet-owned-approval`). For A4 and high-value A7 actions, approvals render in the companion UI (`REQ-ux-companion-ui`) or via WalletConnect to a user-held mobile wallet (TWAK pattern, `research/external/crypto/trust-wallet-agent-kit.md` §9). Novel fields highlighted (`REQ-sec-approval-highlight-changes`).

### 3.6 In-process sequence pool

LaunchTube is being deprecated (`00-context.md#5.4` Track 2); its replacement — OZ Relayer Channels Plugin — is likely AGPL-3.0, clashing with N4 as a deployment dependency. The spec defines an in-process sequence pool (`REQ-classic-sequence-pool`), lifting LaunchTube's deterministic-key-derivation primitive (`research/external/_tier-2/stellar-ecosystem/launchtube.md` §9 item 1) without its third-party-submitter property. Failure-mode semantics under re-org, fee-bump, and `txBadSeq` are spec'd per G7. Throughput is benchmarked (`REQ-perf-sequence-pool-throughput`). Simulation-auth-matches-submission-auth is a spec-required audit (`REQ-sor-simulation-audit`, same source §9 item 3): refuse to submit if footprints diverge, catching a subclass of T7.

### 3.7 Companion UI, Wallet Kit, WalletConnect

The companion UI (`REQ-ux-companion-ui`) is a wallet-owned approval and funding surface, not a CLI re-implementation. It renders approval prompts with the commit nonce, handles funding via Stellar Wallet Kit, and pairs over WalletConnect as a dapp-facing signer. Because it approves the same typed operations as MCP and CLI, it is a UI projection of the same schema. SEP-43 compliance (`REQ-sep-sep43-walletkit`) makes the wallet a Wallet Kit module; WalletConnect v2 with `signAuthEntry` and `signMessage` closes the Creit-WC gap.

The smart-account half of the companion has two concrete reference implementations, both Apache-2.0 and both targeting the OZ `stellar-accounts` v0.7.1 contract the spec's `02-signing-interface` drives. Kalepail's **SAK** is TypeScript and web-only; its demo is a Vite + React SPA with Freighter-browser-extension integration. The Kotlin-Multiplatform **`OZSmartAccountKit`** in `kmp-stellar-sdk` (maintained by Soneso) covers Android, iOS, macOS, and Web; its 7-screen demo integrates Reown / WalletConnect v2 for Freighter-Mobile delegation on mobile. Both produce interoperable contract addresses via a shared default deployer seed (see `research/external/stellar-ecosystem/smart-account-kit.md`).

Under option B's schema-first posture the companion is a UI projection that composes these SDKs: SAK on web, KMP on mobile and native desktop. The wallet's spec-defined approval-channel semantics layer on top (`REQ-sa-companion-sdk-composition`, SHOULD).

### 3.8 Skills / extensibility

Skills add tools without modifying the core (`REQ-ext-skills-pattern`). The manifest (Snaps `initialPermissions` shape, `research/external/_tier-2/crypto/metamask-snaps.md` §9; `REQ-sec-skill-capability-manifest`) declares capability set, tool additions with MCP annotations, `(package, version, shasum)` pinning (`REQ-sec-skill-signed`), publisher signature, and a first-invoke gate on signing (`REQ-sec-skill-first-invoke-gate`, closing Snaps' install-only-consent gap). Skills run isolated from the keystore (`REQ-sec-skill-isolation`); resource caps fail closed (`REQ-perf-resource-caps`); third-party audit is required for signing-touching skills (`REQ-ext-skill-audit-gate`, Rhinestone/Snaps pattern). Because the manifest is schema-driven, a skill's tools appear on every transport — one integration, N transports.

### 3.9 Configuration, identity, secrets

Two-layer config: non-secret TOML at `$XDG_CONFIG_HOME/stellar-ai/` mode `0o700` plus OS-keyring secrets under the same profile name (AWS pattern; `REQ-cfg-two-layer`, `REQ-cfg-xdg`). Env-var overrides with documented precedence (`REQ-cfg-env-vars`, `REQ-cfg-profile-selector`). Non-interactive unlock for CI (`REQ-cfg-noninteractive-unlock`). OS keyring first (`REQ-acct-keyring-first`) with loud `--insecure-storage` opt-out (`REQ-acct-insecure-storage-optout`, gh pattern). Hardware wallet and external-signer plugin as first-class backends. Identities network-scoped (`REQ-sec-network-isolation`); network required on every tool (`REQ-sec-network-required`) and shown on every response (`REQ-sec-network-marker`); no ambient default.

## 4. How this option satisfies the non-negotiables (N1-N6)

**N1 — Self-custodial.** Keys live in OS keyring, on a hardware device, or behind an external-signer plugin (`REQ-acct-keyring-first`, `REQ-acct-hardware-signer`, `REQ-acct-signer-plugin`). The spec's `02-signing-interface.md` forbids `secretKey`-as-tool-argument (`REQ-acct-key-isolation`) — the Stellar MCP anti-pattern (`research/external/stellar-ecosystem/stellar-mcp-server.md` §10). Hardware wallets are normative, not a vendor extension (closing OWS's gap, `research/external/crypto/moonpay-agents-ows.md` §10 item 4).

**N2 — Autonomous operation.** The spec requires end-to-end operation against the user's chosen RPC endpoint with no project-operated service. The in-process sequence pool (§3.6) replaces LaunchTube-class dependencies. Multi-endpoint RPC (`REQ-sec-rpc-multi-endpoint`) lets the user run their own infrastructure. No tool in any transport reaches a project-operated endpoint during signing.

**N3 — No central server.** Configuration, keys, policies, receipts, and audit log live on the user's host. Telemetry off by default (`REQ-tel-no-default`); any opt-in is non-identifying (`REQ-tel-opt-in-non-identifying`).

**N4 — Open source, permissive licence.** Apache-2.0 or MIT for spec, reference implementation, library crate, and reference skills (`REQ-pkg-permissive-licence`); the spec itself CC-BY-4.0 to encourage reimplementation. Safe's LGPL-3.0 misstep and the OZ Relayer AGPL-3.0 risk are avoided by not depending on them.

**N5 — Deterministic, scriptable output.** Structurally enforced by the schema. Every tool has a JSON Schema for its response; MCP delivers verbatim, CLI wraps in the uniform envelope (`REQ-ux-json-envelope`). No per-command drift — the schema is the single source (closing AWS CLI's per-service-drift gap, `research/external/non-crypto/aws-cli.md` §10). Table output is a schema projection, not a separate path.

**N6 — Testnet and mainnet parity.** Every tool accepts a CAIP-2 network parameter; the Stellar chain profile enumerates `stellar:pubnet` and `stellar:testnet` as peers; conformance fixtures cover both. No hardcoded URLs (the Stellar-MCP anti-pattern). Network isolation (`REQ-sec-network-isolation`) keeps testnet and mainnet keystores distinct.

## 5. Per-actor surface (A1-A7)

- **A1 (unattended daemon).** Library (embedded) or MCP (subprocess). Local policy engine with per-tx, per-period, counterparty, and rate-limit rules. Hash-chained audit log (`REQ-sec-audit-log-signed`). Idempotency keys on mutations (`REQ-ux-idempotency-key`). In-process sequence pool.
- **A2 (orchestrator).** C-account with module-attached policy; expiring session signers per sub-agent; synchronous revocation (`REQ-sa-revocation-takes-effect-before-next-sig`). Subaccount derivation (`REQ-acct-subaccounts`) for sub-agents needing independent G-accounts; BIP-44 derivation via `REQ-acct-derivation-sep05` and per-sub-agent independent keypairs via `REQ-acct-subaccount-isolation` (addendum §3).
- **A3 (service consumer).** x402 via `stellar_x402_simulate`/`stellar_x402_commit` (`REQ-svc-x402-consume`, `REQ-svc-x402-nonce-binding`); MPP charge and channel modes via dedicated MPP tool schemas (`REQ-svc-mpp-charge`, `REQ-svc-mpp-channel`, `REQ-svc-mpp-policy-gating`, `REQ-svc-mpp-x402-coexistence`). SEP-10 client with ephemeral auth-only keypairs (`REQ-sep-sep10-ephemeral`). Counterparty allowlists anchored on SEP-10 identity or home domain.
- **A4 (user-facing).** Companion UI as async approval channel (`REQ-ux-companion-ui`); WalletConnect when the user runs a separate mobile wallet. Approval envelopes nonce-bound; novel fields highlighted. Passkey signer on a C-account. OZ `Policy::enforce()` is synchronous-only during `__check_auth` and cannot pause for an out-of-band human ack; the wallet-owned approval channel compensates by collecting the human-approval signal off-chain before submission and embedding it into the auth payload, with a custom policy attached to the rule rejecting the invocation if the signal is absent or stale.
- **A5 (CI/CD).** CLI usually primary. Non-interactive unlock, env-var config, structured logs, dry-run on every mutation. Because the CLI is a schema projection, CI never gets behaviour the spec does not define.
- **A6 (read-mostly).** MCP server with read-only subset only (`REQ-ux-mcp-read-write-groups`); signing unreachable. Reads work without any signing identity (`REQ-obs-no-key-required`). Soroban events as NDJSON on stdout (C6).
- **A7 (human operator, incl. attended development).** CLI with table output, `--help`, setup wizard (`REQ-ux-setup-wizard`). Every operation available to the agent is available to the human with identical semantics. Attended development with IDE agents (Claude Code, Cursor, or similar) inherits the schema's structural network scoping (every tool carries a CAIP-2 network parameter per `REQ-sec-network-required`), hardware-wallet support as a first-class signer kind (`REQ-acct-hardware-signer`), and `simulate` as a peer tool of `send` (`REQ-sor-simulate-first`) so the IDE-agent loop sees a diffable resolved transaction before any signature. Root-quorum break-glass on C-accounts (C4) for recovery.

The human-first property is the hardest one to get right here. The CLI is polished at the presentation layer but its surface is the spec: if a human wants a flag that reorders arguments, that is a presentation-layer addition; if they want behaviour the schema does not describe, that is a spec change.

## 6. Per-threat defences (T1-T10)

**T1 (key exfiltration).** Keyring-first (`REQ-acct-keyring-first`); short unlock windows (`REQ-sec-short-unlock`, Tether WDK `.dispose()` pattern); hardware path; external-signer plugin for TEE/passkey; skill sandbox cannot read the keystore.

**T2 (prompt-injected transaction).** Local policy engine is mandatory spec component (`REQ-sec-local-policy-engine`), evaluating before the signer is reachable. Wallet-issued commit nonce means a prompt-injected retry with `confirmed: true` cannot bypass simulation. Typed tool parameters close the free-form-XDR vector.

**T3 (hallucinated arguments).** JSON Schema on every tool; typed amounts with explicit unit; typed Stellar types; `simulate` as a peer tool; simulation-auth-matches-submission-auth audit; network required on every call.

**T4 (delegated-authority abuse).** C-account with `__check_auth` inspecting `_auth_contexts` (which Meridian Pay notably does not). OZ context-rule-plus-policy delegation (not Safe-style module attachment — OZ does not expose modules). Revocation enforced at the next `__check_auth` (`REQ-sa-revocation-takes-effect-before-next-sig`). Never key sharing (`REQ-sa-delegation-as-policy-not-key`). The auth digest binds `context_rule_ids` (OZ v0.7 hardening: `sha256(signature_payload ‖ context_rule_ids.to_xdr())`), structurally preventing rule-ID downgrade by a malicious sponsor (T11); verifier pinning by wasm hash at install time (`REQ-sa-verifier-pinning`) closes the cross-account verifier replacement vector (T13, T14); atomic signer-and-threshold updates via `ExecutionEntryPoint` (`REQ-sa-atomic-signer-threshold-update`) prevent the signer-set-divergence brick (T12).

**T5 (counterparty impersonation).** Identity-anchored allowlists (home domain, SEP-10 identity, known issuer). Approval prompt displays resolved identity, not just address. Memo-required structurally enforced (`REQ-classic-memo`).

**T6 (runaway loop).** Rate limits, per-period caps, minimum-reserve guard, Soroban resource-fee and footprint caps.

**T7 (RPC tampering).** Multi-endpoint configuration (`REQ-sec-rpc-multi-endpoint`) with `endpoint` field on every response; simulation-auth audit catches a subclass; multi-endpoint cross-check (C10) SHOULD for high-value actions.

**T8 (supply-chain).** Reproducible builds, signed releases, signed skills with `(package, version, shasum)` pinning and audit attestation, skill sandbox, first-invoke gate on signing. Conformance fixtures let a second builder verify the release binary produces identical receipts.

**T9 (approval-channel manipulation).** Wallet-owned approval UX rendered by companion UI, not by the agent. WalletConnect for user-held mobile wallets. Approval envelopes nonce-bound; novel fields highlighted.

**T10 (network confusion).** CAIP-2 required per tool; chain profile enumerated in the spec; `stellar_*` tool-name prefix; network marker on every response; separate keystores per network. The spec makes network confusion structurally impossible: a tool call does not type-check without an explicit network.

## 7. SEP coverage and Soroban auth-entry flows

The Stellar chain profile enumerates: **classic ops** (payments, trustlines, path payments, fee-bump; `REQ-classic-payment`, `REQ-classic-trustline`, `REQ-classic-path-payment`, `REQ-classic-fee-estimate`); **Soroban** (`invokeHostFunction`, deployment, `signAuthEntry` as a distinct operation — `REQ-sor-invoke`, `REQ-sor-deploy`, `REQ-sor-auth-entries` — with simulate as peer of submit); **SEP-10** client with ephemeral keys (`REQ-sep-sep10-client`, `REQ-sep-sep10-ephemeral`); **SEP-43** verbatim from `research/external/stellar-ecosystem/freighter-walletkit.md` §9, with `signAuthEntry` and `signMessage`; **`stellar.toml` resolution** surfacing home-domain and SEP-10 identity on approval prompts (`REQ-sep-stellar-toml`); **x402** handled via Soroban auth entries (a narrow case of the Soroban signing surface, not a separate protocol family); **MPP** (Machine Payments Protocol, IETF individual submission `draft-ryan-httpauth-payment`, covered in `research/stellar-capabilities/10-mpp.md`) as a sibling protocol with its own charge and channel modes expressed as distinct typed tool calls in the schema — charge is a specialised `signAuthEntry` for SAC `transfer`; channel adds a new signing primitive (off-chain cumulative commitments signed with a dedicated ed25519 key) that does not route through the `SorobanAuthorizationEntry` surface and needs a separate schema entry. Requirement anchors: `REQ-svc-mpp-charge`, `REQ-svc-mpp-channel`, `REQ-svc-mpp-commitment-keys`, `REQ-svc-mpp-null-source-account`, `REQ-svc-mpp-challenge-validation`, `REQ-svc-mpp-policy-gating`, `REQ-svc-mpp-receipt-audit`, `REQ-svc-mpp-x402-coexistence`, `REQ-svc-mpp-version-pinning`, `REQ-svc-mpp-mcp-transport`; `REQ-svc-mpp-channel-dispute` blocked on upstream spec.

Out of scope v1: SEP-24 / SEP-6 (`REQ-sep-out-of-scope-issuance`); ships later as a skill without schema churn.

Soroban auth-entry flows are the most Stellar-specific part of the spec. The chain profile defines `SorobanAuthorizationEntry` signing with footprint, resource fees, and per-entry nonce. Signer-shape routing (inspect `needsNonInvokerSigningBy()` to choose classic-keypair vs. C-account, `research/external/stellar-ecosystem/stellar-mcp-server.md` §9) is spec-defined, not per-binary.

## 8. Delta vs. SDF + OZ collaborations (§5.4)

**Track 1 (x402-on-Stellar + x402-MCP server).** The SDF/OZ x402 effort is about agents discovering and paying for HTTP resources. Option B's x402-consume flow is a spec-defined subset of the Soroban signing surface (§7). Where the SDF x402-MCP is a narrow tool, the wallet is the schema the SDF x402-MCP agents can talk to: the spec covers the Soroban auth-entry signing those payments require, and the policy engine provides the spending policy the release narrative names. Positioning: signer-and-policy layer under x402-MCP, not competitor. A cleanly-specified wallet lets multiple paid-API frontends share one custody layer.

**Track 2 (LaunchTube → OZ Relayer Channels Plugin).** The in-process sequence pool (§3.6) is the self-custodial alternative. OZ Relayer requires an operator who sees plaintext envelopes correlated to a JWT subject (`research/external/_tier-2/stellar-ecosystem/launchtube.md` §9). That is the property forcing Meridian Pay and Stellar MCP to fail N2/N3. The spec excludes that operator from the trust base by shipping the sequence-pool primitive in-process. For users who want a shared submission service, OZ Relayer remains available; no longer load-bearing.

Neither track's scope matches option B's. Track 1 is paid API resources; Track 2 is submission infrastructure. The wallet here is the broader self-custodial primitive that includes both as sub-flows. The spec makes coverage gaps and overlaps checkable claims against each track.

## 9. What this option explicitly does not do

- **Does not ship MCP as the only surface.** CLI and library crate are first-class derived transports; A5 and A7 are not second-class.
- **Does not re-implement `soroban_cli`.** Option B reuses the `soroban_cli` crate for Stellar primitives (XDR assembly, simulation, SEP-5 HD paths). What it does not reuse is stellar-cli's command surface: the CLI is generated from the spec, not lifted from stellar-cli's `clap` tree. Honest cost: stellar-cli's external-binary plugin pattern (`stellar-<name>` on `PATH`) does not come for free; a manifest-based skill system replaces it.
- **Does not build an LLM into the wallet.** The wallet is the subject of agent tool calls, not an agent.
- **Does not depend on LaunchTube or OZ Relayer.** The in-process sequence pool removes that dependency; callers may still configure a relayer, but it is not in the trust base.
- **Does not ship x402-server primitives in v1** (`REQ-svc-x402-out-provide`); SEP-24/SEP-6 in v1 (`REQ-sep-out-of-scope-issuance`); Windows in v1 (`REQ-plat-windows`).
- **Does not provide `raw_sign`** over opaque bytes (C9). Policy cannot reason about opaque bytes; the spec forbids it.

## 10. Implementation cost estimate (rough)

Components split into "spec plus generator" (new work) and "reused" (shared with option A).

**New work — spec + generators (option-B-specific):**

1. **Spec drafting.** Seven sub-spec documents plus Stellar chain profile. ~8-12 weeks lead-dev to draft; ~4-6 weeks iteration from internal and external review.
2. **Schema-driven generator.** JSON Schema → Rust/TS/Python types + CLI `clap` definitions + MCP tool registrations. ~6-8 weeks for the first working generator; continuous refinement as the spec evolves.
3. **Conformance test suite.** Golden fixtures per tool, per chain profile. ~4-6 weeks initial; every new tool adds fixtures.
4. **Version negotiation layer.** Spec-version advertisement, intersection, migration notes. ~2 weeks.
5. **WalletConnect integration with `signAuthEntry`/`signMessage` + SEP-43 Wallet Kit module.** ~5-7 weeks combined.

**Shared with option A (reusable as-is):** key handling (keyring, hardware wallet, external-signer plugin); policy engine implementation (option B specifies the JSON grammar up front, option A evolves it — code is shared); smart-account contract template; in-process sequence pool; hash-chained audit log; companion UI screens; skill system (manifest, sandbox, signed install — option B makes the manifest schema normative).

**Honest comparison to option A.** Option B's additional upfront cost concentrates in points 1-4: roughly 20-30 weeks of lead-dev before the first CLI demo. Option A reaches a usable CLI faster because it reuses stellar-cli's surface and grows MCP alongside. Option B's payoff arrives later: when third parties write conformance-tested reimplementations, when multiple x402-MCP / Wallet Kit / custom frontends route to the same custody. That payoff is second-year, not first-quarter.

## 11. Primary risks

1. **Spec lock-in.** Once the schema is public and third parties target it, breaking changes cost. Mitigation: version-negotiated protocols; keep v0 explicitly experimental for six months before freezing v1.
2. **Generator complexity.** A poor generator produces a mechanical CLI and awkwardly-named MCP tools. Mitigation: the generator is a first-class deliverable, tested against the same conformance suite as any other transport.
3. **Schema vs. reality drift in skills.** A skill may want a tool shape the generator does not support. Mitigation: the manifest is the schema entry point; skills that exceed the generator model either extend the generator upstream or live as CLI-only plugins (degradation path).
4. **Conformance-testing burden.** Every new tool adds fixtures per transport. Mitigation: fixtures generated from a single golden file per tool; per-transport validation runs automatically.
5. **Adoption risk.** If no third party ever reimplements the spec, the conformance infrastructure is lower-leverage than promised. Mitigation: the suite still protects the reference implementation from regression.
6. **Upfront velocity.** First demoable CLI lands later than option A. RFP reviewers may see this as a red flag unless the spec itself is a presented deliverable.
7. **Stellar protocol evolution.** Protocol additions that do not fit the chain profile cleanly trigger profile updates. The spec is a clear current state, not eternal.

## 12. Honest self-assessment

Option B is weaker than option A in several places RFP reviewers should see named:

1. **Higher upfront cost for first demo.** Option A reaches a usable CLI earlier by reusing stellar-cli's command surface and growing MCP beside it. Option B's first CLI lands later because the surface is generated from the spec, and the spec comes first. If the success criterion is "working CLI soon," A wins on tempo.
2. **Lost `soroban_cli` direct-reuse on the CLI surface.** Option B still depends on the `soroban_cli` crate for Stellar primitives, but does not inherit its `clap`-generated command tree. Option A does. That is a real ergonomic win for users who already know `stellar contract invoke`. Option B's CLI will resemble stellar-cli without being it; operators will notice.
3. **CLI UX is derived rather than crafted.** A human operator who wants a flag tailored to a specific workflow faces a schema change rather than a CLI patch. Fine in steady state, awkward during early iteration.
4. **Extending `stellar-cli` directly is not on the table.** Option A has a clean upstream-contribution path to SDF's incumbent. Option B's contribution path is the spec itself — slower to influence, harder to frame as "improved the incumbent."
5. **Spec complexity is itself a cost.** Seven sub-specs add cognitive overhead. OWS has the same cost and a 20-backing-org adoption path; option B does not start with that.
6. **Generator maintenance is never over.** Every new tool, schema refinement, and language binding is generator work — a running cost.
7. **Third-party reimplementation may never happen.** The central promise — that a community implementation passes the same conformance suite — is real but speculative. The suite still guards the reference implementation from regression, so it is not fully sunk.

Where option B is stronger: T2/T3/T9/T10 defences are structural in the schema rather than conventions in the CLI; wallet-issued commit nonces baked in from day one, closing the advisory-only `confirmed: false → true` anti-pattern at the protocol level (see `_tier-2/crypto/phantom-mcp-server.md` §10 and `_summary.md` cross-cutting finding 5); an Nth surface costs a generator run not a new repo; conformance is automatable, so RFP reviewers can run golden fixtures themselves; positioning relative to SDF tracks 1 and 2 is crisp because schema coverage/overlap are checkable claims.

Do not strawman option A. Depending on the RFP's weighting of time-to-first-demo vs. long-term third-party implementability, A may be the correct choice. This doc argues B's case; `09-decision-matrix.md` and `10-recommendation.md` weigh them.

## 13. Cross-references to the companion option

Option A (`06-option-a-cli-first.md`, drafted in parallel) centres on extending stellar-cli's command surface with agent affordances — MCP wrapper, typed amounts, policy engine — layered onto the incumbent CLI. A and B share every component marked "shared with option A" in §10: key handling, policy engine implementation, smart-account template, sequence pool, audit log, companion UI, skill system. Where they differ is the centre of gravity:

- **Versioned and advertised artefact.** A: the binary and its `--help`. B: the spec and its schemas.
- **Conformance-tested artefact.** A: integration tests against the binary. B: golden fixtures against any implementation.
- **"Add a feature."** A: edit the `clap` tree, mirror in the MCP wrapper. B: edit the spec, regenerate all transports.
- **What a third party reaches for.** A: the CLI, directly. B: the MCP server for agents, CLI for humans, the spec to reimplement.

Option B is more ambitious; option A more tractable. The analysis doc answers which ambition best matches the RFP award horizon, the evolving SDF x402-MCP and OZ Relayer surfaces, and the actor mix expected in years 1-2.

### Requirement citations

REQ-IDs cited inline above, grouped by area: **account/keys** `REQ-acct-{key-isolation, keyring-first, insecure-storage-optout, hardware-signer, signer-plugin, subaccounts}`; **classic+Soroban** `REQ-classic-{payment, memo, path-payment, sequence-pool}`, `REQ-sor-{invoke, simulate-first, auth-entries, deploy, simulation-audit, footprint-cap}`; **smart-account** `REQ-sa-{c-account-support, policy-initiated-exec, webauthn-signer, revocation-takes-effect-before-next-sig, delegation-as-policy-not-key, session-keys, guard-hooks}`; **SEP / agentic payments** `REQ-sep-{sep10-client, sep10-ephemeral, sep43-walletkit, stellar-toml, out-of-scope-issuance}`, `REQ-svc-{x402-consume, x402-nonce-binding, x402-out-provide, mpp-charge, mpp-channel, mpp-commitment-keys, mpp-policy-gating, mpp-x402-coexistence, mpp-mcp-transport}`; **read** `REQ-obs-{no-key-required, events-stream}`; **security/policy** `REQ-sec-{local-policy-engine, policy-per-tx-cap, policy-per-period-cap, policy-counterparty-allowlist, policy-rate-limit, policy-minimum-reserve, policy-signed, policy-mutation-governance, dangerous-tool-gate, typed-amounts, typed-amount-errors, network-required, network-marker, network-isolation, rpc-multi-endpoint, wallet-owned-approval, approval-highlight-changes, approval-nonce, skill-isolation, skill-signed, skill-capability-manifest, skill-first-invoke-gate, audit-log-signed, short-unlock}`; **UX/MCP** `REQ-ux-{json-default, json-envelope, table-render, query-projection, exit-codes, error-taxonomy, idempotency-key, mcp-stdio, mcp-tool-annotations, mcp-tool-names-namespaced, mcp-read-write-groups, setup-wizard, dry-run, companion-ui}`; **audit/perf/ext/pkg/cfg** `REQ-audit-{stderr-log, policy-denials}`, `REQ-perf-{sequence-pool-throughput, resource-caps}`, `REQ-ext-{skills-pattern, skill-audit-gate, library-crate}`, `REQ-pkg-{reproducible-builds, signed-releases, permissive-licence}`, `REQ-plat-windows`, `REQ-cfg-{two-layer, xdg, env-vars, profile-selector, noninteractive-unlock}`, `REQ-tel-{no-default, opt-in-non-identifying}`. Gaps and candidates: G1, G3, G4, G7, C1, C4, C6, C9, C10.

### Research citations

Primary sources (each cited inline with specific §-item references above):

- `research/external/_summary.md` — strategic findings 4, 5, 6; Non-wallet findings (Creit WC gaps).
- `research/external/crypto/moonpay-agents-ows.md` — §9 items 1, 3, 6; §10 items 1, 4.
- `research/external/crypto/kraken-cli.md` — §9; `research/external/crypto/trust-wallet-agent-kit.md` — §9; `research/external/crypto/safe.md` — §9; `research/external/crypto/turnkey.md` — §9; `research/external/crypto/coinbase-agentic.md` — §9.
- `research/external/stellar-ecosystem/meridian-pay.md` — §9 item 2; §10 item 1. `research/external/stellar-ecosystem/stellar-mcp-server.md` — §9, §10. `research/external/stellar-ecosystem/freighter-walletkit.md` — §9. `research/external/stellar-ecosystem/stellar-cli.md` — §9.
- `research/external/non-crypto/{aws,gh,stripe}-cli.md` — §9 (two-layer config, `credential_process`, keyring-first, idempotency, `listen --forward-to`).
- `research/external/_tier-2/crypto/phantom-mcp-server.md` — §9 items 1-5; §10 items 2, 5. `research/external/_tier-2/crypto/tether-wdk.md` — §9 items 1, 3, 4; §10 items 1, 2. `research/external/_tier-2/crypto/metamask-snaps.md` — §9, §10. `research/external/_tier-2/crypto/privy.md` — §9, §10. `research/external/_tier-2/crypto/fireblocks.md` — §9.
- `research/external/_tier-2/stellar-ecosystem/launchtube.md` — §9 items 1, 3; §10.
