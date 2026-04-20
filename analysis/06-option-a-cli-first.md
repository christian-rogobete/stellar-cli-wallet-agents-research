# Option A: CLI-first with agent affordances

**Status:** draft — 2026-04-18
**Audience:** RFP Track delegates
**Depends on:** `00-context.md`, `01-actor-model.md`, `02-threat-model.md`, `03-requirements.md`, `research/external/_summary.md`

---

## 1. Definition

Option A designates the **CLI as the canonical surface** and all other surfaces (MCP, library, companion UI) as **derivations of that CLI**. Concretely, the wallet is a single Rust binary on `PATH` — working name `stellar-agent-wallet` — that:

- registers as an external-binary plugin under the incumbent `stellar` command (any `stellar-<name>` on `PATH` becomes `stellar <name>`; `stellar-cli.md` §9),
- embeds the public `soroban_cli` library crate for identity files, SEP-5 HD derivation, secret format, OS-keyring and Ledger signers, Soroban tx assembly, XDR plumbing (`stellar-cli.md` §9),
- ships an MCP stdio server as a **second transport over the same dispatch path** — one command tree, two transports, schemas from clap metadata — Kraken-CLI pattern (`kraken-cli.md` §9),
- adds what `stellar-cli` is missing: local policy engine, C-account signer, uniform `{ok, data|error, request_id}` JSON envelope, wallet-owned approval channel, in-process sequence pool, typed amount/identity schema (`stellar-cli.md` §10).

The companion UI — an optional desktop / mobile application reached via SEP-43, Stellar Wallets Kit, and WalletConnect v2 — is **not** part of the primary surface; it is one channel through which a user approves pending actions.

## 2. Centre of gravity and what that implies

Option A claims that the CLI is where the wallet is **authored, tested, versioned and documented**; every other surface is generated from or dispatched into it. The MCP server's tool list is the CLI command list, its schemas are the clap parameter definitions, its handlers are the same functions the CLI subcommands call. Adding a capability means adding a subcommand; it appears as an MCP tool automatically.

This is a claim about **provenance**, not about which surface callers use. A large fraction of agent traffic is expected over MCP; human and scripted CI traffic over the CLI; high-touch human approval over the companion UI. Option A is not "the CLI is the only surface"; it is "the CLI's typed parameter surface is the source of truth, and everything else is structurally downstream."

What is **not** implied by CLI-first:

- **Not** that agents shell out and parse stdout. Agents use MCP; shelling out with `--json` is a supported fallback.
- **Not** that the companion UI is an afterthought. Wallet-owned approval (REQ-sec-wallet-owned-approval; T9) is downstream in provenance but essential in function.
- **Not** that MCP is deferred. MCP ships in v1 (REQ-ux-mcp-stdio is MUST). What is canonical is the **command surface**, not `main()`.
- **Not** that library embedders are second-class. The library crate is an explicit artifact (REQ-ext-library-crate); the CLI is a thin shell over internal crates that can be imported directly.

## 3. Concrete architecture

### 3.1 Primary binary and extension strategy

The wallet ships as `stellar-agent-wallet`: a Rust binary installed via `cargo install`, `brew`, and pre-built GitHub release binaries (REQ-pkg-prebuilt-binaries) for macOS and Linux arm64 / x86_64 (REQ-plat-macos, REQ-plat-linux). Placed on `PATH`, it is discovered by `stellar-cli` via the external-binary plugin pattern (`stellar-cli.md` §9): `stellar agent-wallet <subcommand>` dispatches into the wallet binary. Users without `stellar-cli` use `stellar-agent-wallet` directly.

Three invocation modes:

- `stellar-agent-wallet <subcommand> …` — CLI returning `{ok, data|error, request_id}` JSON by default (REQ-ux-json-default, REQ-ux-json-envelope), `--output table` (REQ-ux-table-render), `--query` projection (REQ-ux-query-projection).
- `stellar-agent-wallet mcp [--tools read|write|all] [--network …]` — MCP stdio server (REQ-ux-mcp-stdio) with read/write groupings (REQ-ux-mcp-read-write-groups), chain-namespaced tool names (REQ-ux-mcp-tool-names-namespaced), and `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` annotations (REQ-ux-mcp-tool-annotations).
- `stellar-agent-wallet setup` — interactive wizard for identity, keyring, default profile (REQ-ux-setup-wizard, REQ-acct-keyring-first).

Three internal crates:

1. `stellar-wallet-core` — embeds `soroban_cli` as a library dep (identity files, SEP-5, secret format, Ledger, OS-keyring; `stellar-cli.md` §9), extends `Signer` / `SignerKind` with C-account and passkey backends (closing the `ScAddress::Contract` gap, `stellar-cli.md` §5), ships policy engine, sequence pool, JSON envelope, audit log.
2. `stellar-wallet-cli` — clap-based CLI over `-core`; the command tree is the source of truth for MCP schemas.
3. `stellar-wallet-mcp` — `rmcp`-based MCP server whose tool list is derived from clap metadata (Kraken pattern, `kraken-cli.md` §9). No subprocess wrappers; no hand-maintained schema.

This structure delivers REQ-ext-library-crate by construction: downstream consumers depend on `stellar-wallet-core` directly and skip the CLI shell.

### 3.2 Policy engine

The policy engine lives in `stellar-wallet-core` and is evaluated **before the signer is touched** on every signing attempt, regardless of transport (REQ-sec-local-policy-engine; `02-threat-model.md` §6 implication 1; T2/T4/T6).

Vocabulary is the **union** of patterns converged across the strongest agent-oriented products (`_summary.md` cross-cutting finding 2):

- **Typed criteria** (Coinbase CDP, `coinbase-agentic.md` §9): per-tx caps in USD (`changeCents`) and native units (REQ-sec-policy-per-tx-cap); per-period caps (REQ-sec-policy-per-period-cap); rate limits (REQ-sec-policy-rate-limit); Soroban resource-fee caps (REQ-sor-footprint-cap).
- **Typed counterparty object kinds** (Fireblocks, `_tier-2/crypto/fireblocks.md` §9): `G_ACCOUNT` / `C_ACCOUNT` / `SEP10_IDENTITY` / `HOME_DOMAIN` / `KNOWN_ISSUER` / `ONE_TIME_ADDRESS`, identity-anchored (REQ-sec-policy-counterparty-allowlist; T5).
- **Token-as-capability via HKDF re-wrap** (MoonPay OWS, `moonpay-agents-ows.md` §9): scoped tokens are HKDF inputs to re-derive decryption material, so revoking a token revokes decryption (T1, T4); used for sub-agent budget grants (REQ-sa-delegation-as-policy-not-key).
- **Guard hooks** (Safe, `safe.md` §9): pre-sign / post-execution on the C-account (REQ-sa-guard-hooks).
- **Fail-closed executable policy plugin** with 5-second timeout (MoonPay OWS escape hatch) for rules the declarative layer cannot express; shipped as a skill, not the first answer.
- **Dangerous-tool gate with `acknowledged=true`** + fail-closed argv filter (Kraken, `kraken-cli.md` §9; REQ-sec-dangerous-tool-gate).

Policy files are **signed by the owner identity** (REQ-sec-policy-signed; `_tier-2/crypto/privy.md` §9 `owner_id`); mutations require owner signature (REQ-sec-policy-mutation-governance); denials are logged with the draft they refused (REQ-audit-policy-denials). The minimum-reserve guard refuses to cross `2 + (base_reserve * subentries)` XLM (REQ-sec-policy-minimum-reserve; pending G3).

### 3.3 Smart-account delegation

A C-account template ships with the wallet and can be deployed per user, per scoped sub-agent, per CI-deploy context, or per x402 counterparty cohort. The template starts from the ~100 LOC Meridian Pay `__check_auth` WebAuthn baseline (`meridian-pay.md` §9) and extends it with policy inspection of `_auth_contexts`, rather than ignoring it as Meridian Pay does (`meridian-pay.md` §4, §10).

The C-account supports:

- **G-account and C-account as signer sources** (REQ-sa-c-account-support), closing the `stellar-cli` `ScAddress::Contract` gap (`stellar-cli.md` §5).
- **Passkey (WebAuthn) signer kind** (REQ-sa-webauthn-signer), reusing Meridian Pay's `webauthn::verify`.
- **Multiple signer kinds per account** via the OZ `stellar-contracts` accounts pattern rather than Meridian Pay's single-signer shape (`meridian-pay.md` §13).
- **Custom wallet machinery for recurring-payment and top-up patterns** (REQ-sa-policy-initiated-exec). OZ v0.7.1 `Policy` objects can only accept or revert inside `__check_auth` — they cannot originate transactions. Recurring-payment and top-up flows are therefore **custom work the wallet ships on top of OZ context rules**, not an OZ-provided extension of the `Policy` trait: a wallet-owned scheduler or keeper signs the next invocation under a dedicated context rule whose policy caps amount, counterparty and period (`safe.md` §9 discusses the corresponding Safe-module pattern, which OZ does not expose).
- **Expiring session keys** (REQ-sa-session-keys; Turnkey, `turnkey.md` §9): 15 min default, non-exportable, revocation enforced at the next `__check_auth` call so newly-constructed signatures are rejected (REQ-sa-revocation-takes-effect-before-next-sig). OZ rule lookup happens inside `__check_auth`, not at signing time, so revocation is a forward-looking guarantee — already-signed envelopes in flight can still succeed if submission races the revocation ledger.
- **Router / multicall contract** (C11; `meridian-pay.md` §9) to bundle invocations under one approval (T9 habituation defence).
- **Deterministic address derivation from salt** (`meridian-pay.md` §9), combined with OZ `stellar-merkle-distributor` for pre-funded bounded inbound budgets.

Delegation is expressed as **scoped on-chain policy, scoped Soroban auth entry, or scoped subaccount**, never shared key material (REQ-sa-delegation-as-policy-not-key; `02-threat-model.md` §6, implication 4).

### 3.4 MCP surface as derived transport

The MCP stdio server is a **second transport over the CLI dispatch path**, not a parallel implementation (REQ-ux-mcp-stdio; Kraken-CLI, `kraken-cli.md` §9: "one command tree, two transports; no subprocess wrappers; schemas generated from clap metadata"). Concretely:

- Each subcommand's clap derivation produces both the CLI parser and the MCP tool schema; no hand-maintained duplicate.
- Annotations (`readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint`) come from attribute macros (REQ-ux-mcp-tool-annotations; `_tier-2/crypto/tether-wdk.md` §9, `_tier-2/crypto/phantom-mcp-server.md` §9).
- Tool names are chain-namespaced (`stellar_payment_send`, `stellar_contract_invoke`) with CAIP-2 on the network field (REQ-ux-mcp-tool-names-namespaced; T10).
- Read/write split is a registration-time flag (`--tools read`); the write tool is absent from the read schema (REQ-ux-mcp-read-write-groups; `_tier-2/crypto/tether-wdk.md` §9; serves A6).
- Amount fields use a dual-unit schema (`amountUnit` + `decimals`; REQ-sec-typed-amounts; `_tier-2/crypto/phantom-mcp-server.md` §9) with typed parse errors (REQ-sec-typed-amount-errors).
- Two-step simulate-then-commit where `commit` binds to a **wallet-issued nonce returned by `simulate`**, not an agent boolean (REQ-svc-x402-nonce-binding; fixes `_tier-2/crypto/phantom-mcp-server.md` §10 anti-pattern; `_summary.md` cross-cutting finding 5). Closes the advisory-only `confirmed: false → true` anti-pattern at the protocol level.
- Errors carry a typed `category` enum (REQ-ux-error-taxonomy; Kraken nine-category, `kraken-cli.md` §9); CLI exit codes distinct per class (REQ-ux-exit-codes).

The MCP server honours the **same policy engine** as the CLI: one evaluator, one pass per signing attempt, independent of transport.

### 3.5 In-process sequence pool

Option A ships an **in-process sequence pool as a first-class feature**, not a fallback (REQ-classic-sequence-pool; `_summary.md` tier-2 finding 4). The primitive is lifted from LaunchTube (`launchtube.md` §9): deterministic `SHA-256(master || index)` pool-key derivation, credit-as-capability accounting with eager → bid → final spend, simulation-auth-matches-submission-auth audit (REQ-sor-simulation-audit).

Strategic context (`00-context.md` §5.4): LaunchTube is being **discontinued**; OZ Relayer Channels Plugin is the replacement (licence likely AGPL-3.0 — clash with N4 as a deployment dependency); and a third-party submitter that sees plaintext envelopes correlated to a bearer token fails N2/N3 regardless of who operates it (`launchtube.md` §10).

Shipping in-process removes the dependency, sharpens positioning relative to SDF track 2, and gives A1 / A2 / A3 / A5 bounded-throughput submission that cannot leak envelopes to a third party.

### 3.6 Companion UI, Wallet Kit, WalletConnect

The companion UI is a separately-shipped optional application — desktop on macOS and Linux, mobile later — serving two purposes:

- **Wallet-owned approval rendering** for high-value or policy-escalated actions (REQ-sec-wallet-owned-approval, REQ-sec-approval-highlight-changes, REQ-sec-approval-nonce). It receives requests from `stellar-agent-wallet` over a local IPC channel, renders UI the agent cannot forge, and returns signed approvals bound to wallet-issued nonces.
- **Funding and interoperability via SEP-43** (REQ-sep-sep43-walletkit): registers as a Stellar Wallets Kit `ModuleInterface` (`freighter-walletkit.md` §9) and ships a WalletConnect v2 client on `stellar:pubnet` / `stellar:testnet` supporting `stellar_signXDR`, `stellar_signAndSubmitXDR`, **plus `signAuthEntry` and `signMessage`** — filling the gap Creit's current WalletConnect module leaves open (`freighter-walletkit.md` §9, §10).

**Reference implementations exist for the smart-account half — with different platform reach.** Two Apache-2.0 client SDKs target the same OZ `stellar-accounts` v0.7.1 contract and produce interoperable contract addresses via a shared default deployer seed (`smart-account-kit.md` §1, §9). Kalepail's TypeScript **SAK** is browser-only; its demo is a Vite + React SPA covering the full lifecycle with Freighter-browser-extension integration. The Kotlin-Multiplatform **`OZSmartAccountKit`** in `kmp-stellar-sdk` (maintained by Soneso) runs on Android, iOS, macOS, and Web; its 7-screen `smart-account-demo` integrates Reown / WalletConnect v2 so Freighter-Mobile can act as a delegated signer on mobile. A production companion composes these SDKs: SAK for a web SPA path if shipped, KMP for native mobile, native desktop, or a shared code path on web. The wallet-owned approval-channel logic, the SEP-43 adapter, and the WalletConnect `signAuthEntry` / `signMessage` integration remain additional work (`REQ-sa-companion-sdk-composition`).

The companion is **not** the center of gravity. It holds no keys, evaluates no policy, talks to no RPC directly. Headless A1 / A5 hosts run without it.

### 3.7 Skills / extensibility

Skills extend the CLI — and therefore the MCP surface — without forking the core (REQ-ext-skills-pattern). Two channels:

- **`stellar-<name>` external binaries on `PATH`** (`stellar-cli.md` §9): incumbent pattern, option-A's lever.
- **In-process sandboxed modules** (REQ-sec-skill-isolation; MetaMask Snaps `initialPermissions` / SES Compartment, `_tier-2/crypto/metamask-snaps.md` §9): per-skill private storage, mandatory `maxRequestTime`, declarative capability manifest (REQ-sec-skill-capability-manifest), first-invoke gate on signing (REQ-sec-skill-first-invoke-gate).

Skills install by **`(package, version, shasum)` pinning + signed attestations** (REQ-sec-skill-signed; `safe.md` §9 Rhinestone; `_tier-2/crypto/metamask-snaps.md` §9 signed-WASM). Skills requesting signing or key-derivation require a **third-party audit attestation** (REQ-ext-skill-audit-gate). No skill reads the keystore, modifies an in-flight transaction, or mutates policy silently. Per-skill CPU / memory caps fail closed (REQ-perf-resource-caps), rejecting the `endowment:long-running` anti-pattern (`_tier-2/crypto/metamask-snaps.md` §10).

### 3.8 Configuration, identity, secrets

Two-layer config (REQ-cfg-two-layer; `aws-cli.md` §9, `stellar-cli.md` §9): non-secret TOML under `$XDG_CONFIG_HOME/stellar-agent-wallet/` at `0o700` (REQ-cfg-xdg); OS-keyring secrets keyed by profile name (REQ-acct-keyring-first). Plaintext only under the named `--insecure-storage` opt-out (REQ-acct-insecure-storage-optout; `gh-cli.md` §9). Precedence: flag > env > profile-file > default (REQ-cfg-env-vars). `STELLAR_WALLET_PROFILE` as named-profile selector (REQ-cfg-profile-selector).

Signer kinds (extending `soroban_cli`'s `Signer` / `SignerKind` enum):

1. `OsKeyring` — default (macOS Keychain / Linux Secret Service; Windows DPAPI post-v1).
2. `Ledger` — hardware (REQ-acct-hardware-signer).
3. `ExternalProcess` — `credential_process`-style plugin (REQ-acct-signer-plugin; `aws-cli.md` §9). Absorbs passkey, TEE, HSM, and future kinds.
4. `CAccount` — Soroban smart-account signer (new; closes the `ScAddress::Contract` gap).
5. `EphemeralAuth` — SEP-10 auth-only keypair, unfunded, unpersisted (REQ-sep-sep10-ephemeral).

Non-interactive unlock (REQ-cfg-noninteractive-unlock) via passphrase file or documented env var; short, named unlock windows (REQ-sec-short-unlock; `_tier-2/crypto/tether-wdk.md` §9 `.dispose()` pattern). Testnet and mainnet identities live in **distinct keystore namespaces** (REQ-sec-network-isolation).

## 4. How this option satisfies the non-negotiables (N1-N6)

- **N1 (self-custodial).** Keys on-host, OS keyring by default, hardware-wallet and smart-account passkey paths available; no server liveness for any signing. Inherits `stellar-cli`'s N1 posture (`stellar-cli.md` §8) and strengthens it by inverting the `keys generate` default (plaintext-on-disk is not default; `stellar-cli.md` §10).
- **N2 (autonomous).** No project-operated backend. In-process sequence pool (§3.5) removes the LaunchTube / OZ-Relayer dependency. Direct user-configured Horizon / Soroban RPC (REQ-sec-rpc-multi-endpoint).
- **N3 (no central server).** Config, identity, policy, audit log, receipts all on-host. Telemetry off by default (REQ-tel-no-default); any opt-in telemetry is non-identifying (REQ-tel-opt-in-non-identifying).
- **N4 (permissive licence).** Apache-2.0 throughout (REQ-pkg-permissive-licence). Comparators failing N4 (Safe LGPL-3.0; OZ Relayer Channels Plugin AGPL-3.0-suspected) inform the avoidance list.
- **N5 (JSON-default).** Uniform `{ok, data|error, request_id}` envelope on every command (REQ-ux-json-default, REQ-ux-json-envelope). Largest upgrade over `stellar-cli`, which emits JSON on only ~9 of ~50 commands (`stellar-cli.md` §8, §10).
- **N6 (testnet/mainnet parity).** Network required on every call (REQ-sec-network-required), shown on every response (REQ-sec-network-marker), distinct keystore namespaces (REQ-sec-network-isolation). Explicitly rejects the `stellar-mcp` hardcoded-testnet anti-pattern (`stellar-mcp-server.md` §10).

## 5. Per-actor surface (A1-A7)

- **A1 (unattended daemon).** Non-interactive unlock, structured stderr audit log (REQ-audit-stderr-log), idempotency on every mutation (REQ-ux-idempotency-key), per-tx / per-period / per-counterparty / rate-limit policy, minimum-reserve guard, sequence pool. MCP for MCP-speaking agents; CLI for the rest.
- **A2 (multi-agent orchestrator).** Subaccount derivation (REQ-acct-subaccounts), salt-derived addresses, scoped C-account per sub-agent, expiring session keys, synchronous revocation, policy-initiated execution, sequence-pool throughput.
- **A3 (service-consumer, x402 / MPP / SEP-10).** SEP-10 client (REQ-sep-sep10-client) with ephemeral auth-only keypairs (REQ-sep-sep10-ephemeral), x402 consume (REQ-svc-x402-consume) with wallet-issued nonce binding (REQ-svc-x402-nonce-binding), correlated receipts (REQ-svc-x402-receipt); MPP charge and channel modes (REQ-svc-mpp-charge, REQ-svc-mpp-channel, REQ-svc-mpp-policy-gating, REQ-svc-mpp-x402-coexistence); identity-anchored allowlists (REQ-sec-policy-counterparty-allowlist).
- **A4 (user-facing assistant).** Companion UI renders approvals; WalletConnect v2 closes the mobile path. Multicall router bundles invocations; novel-field highlighting (REQ-sec-approval-highlight-changes); passkey signer on the C-account. OZ `Policy::enforce()` is synchronous-only during `__check_auth` — it cannot pause for an out-of-band human ack. The wallet-owned approval channel compensates: the human-approval signal is collected off-chain before submission and embedded into the auth payload, and a custom policy attached to the rule rejects the invocation if the signal is absent or stale.
- **A5 (CI/CD deploy).** Non-interactive unlock, dry-run, idempotency, structured audit log, `credential_process`-style external signer for OIDC-issued short-lived credentials, scoped deploy C-account per repo.
- **A6 (read-mostly research).** Read-path works without a key (REQ-obs-no-key-required); `stellar-agent-wallet mcp --tools read` exposes only read tools (REQ-obs-account-state, REQ-obs-events-stream).
- **A7 (human operator, incl. attended development).** Everything agents can do, humans do more directly: CLI with table output, setup wizard, audit-log queries by hash or date range (REQ-audit-receipts-human-retrievable), hash-chained log. The attended-dev path inherits `stellar <subcommand>` discovery via the external-binary plugin pattern (`stellar-cli.md` §9); structural network scoping per call (REQ-sec-network-required) so a misconfigured default cannot ship a mainnet command by accident; hardware-wallet first-class (REQ-acct-hardware-signer); dry-run (REQ-ux-dry-run) and `simulate` as a peer tool of `invoke` (REQ-sor-simulate-first) so the IDE-agent loop sees a diffable resolved transaction before any signature. The CLI is A7's native surface; this is the actor option A serves best.

## 6. Per-threat defences (T1-T10)

- **T1 (key exfiltration).** Keyring-first (REQ-acct-keyring-first), hardware signer (REQ-acct-hardware-signer), external-signer plugin (REQ-acct-signer-plugin), short unlock windows (REQ-sec-short-unlock), HKDF-bound scoped tokens (`moonpay-agents-ows.md` §9), subaccounts so exfil is not total loss.
- **T2 (prompt-injected transaction).** Local policy engine evaluated before signer touched (REQ-sec-local-policy-engine); typed amounts (REQ-sec-typed-amounts); wallet-owned approval (REQ-sec-wallet-owned-approval); dangerous-tool gate (REQ-sec-dangerous-tool-gate); C-account `__check_auth` inspects `_auth_contexts` (fixing the Meridian Pay gap, `meridian-pay.md` §4).
- **T3 (hallucinated arguments).** Typed MCP schemas from clap, typed amount parse errors, simulate-first peer tool (REQ-sor-simulate-first), simulation-auth-matches-submission-auth (REQ-sor-simulation-audit), network required per call.
- **T4 (delegated-authority abuse).** Delegation is policy or auth-entry, never key (REQ-sa-delegation-as-policy-not-key); synchronous revocation at the next `__check_auth` (REQ-sa-revocation-takes-effect-before-next-sig); expiring session keys (REQ-sa-session-keys); HKDF-bound tokens. The auth digest binds `context_rule_ids` (OZ v0.7 hardening: `sha256(signature_payload ‖ context_rule_ids.to_xdr())`), structurally preventing rule-ID downgrade by a malicious sponsor (T11); verifier pinning by wasm hash at install time (REQ-sa-verifier-pinning) closes the cross-account verifier replacement vector (T13, T14); atomic signer-and-threshold updates via `ExecutionEntryPoint` (REQ-sa-atomic-signer-threshold-update) prevent the signer-set-divergence brick (T12).
- **T5 (counterparty impersonation).** Identity-anchored allowlists (REQ-sec-policy-counterparty-allowlist), stellar.toml resolution on approval (REQ-sep-stellar-toml), Fireblocks-style typed object kinds (`_tier-2/crypto/fireblocks.md` §9), memo-required enforcement (REQ-classic-memo).
- **T6 (runaway loop).** Rate limits (REQ-sec-policy-rate-limit), per-period caps (REQ-sec-policy-per-period-cap), minimum-reserve guard (REQ-sec-policy-minimum-reserve), Soroban resource-fee caps (REQ-sor-footprint-cap), per-skill CPU / memory caps (REQ-perf-resource-caps).
- **T7 (RPC tampering).** Multi-endpoint (REQ-sec-rpc-multi-endpoint), endpoint field on every response, simulation-auth-audit catches footprint modification between simulate and submit.
- **T8 (supply-chain).** Reproducible builds (REQ-pkg-reproducible-builds), signed releases (REQ-pkg-signed-releases), skill `(package, version, shasum)` pinning + signatures (REQ-sec-skill-signed), audit-gate on signing capabilities (REQ-ext-skill-audit-gate), capability manifest (REQ-sec-skill-capability-manifest), hash-chained audit log.
- **T9 (approval-channel manipulation).** Approval UX is wallet-controlled (REQ-sec-wallet-owned-approval); novel-field highlighting (REQ-sec-approval-highlight-changes); wallet-issued nonces (REQ-sec-approval-nonce); multicall reduces habituation; WalletConnect as approval transport (`trust-wallet-agent-kit.md` §9).
- **T10 (network confusion).** Network required on every call, shown on every response, separate keystore namespaces, chain-namespaced MCP names with CAIP-2. Structurally impossible.

## 7. SEP coverage and Soroban auth-entry flows

Option A covers in v1: **SEP-5** HD derivation (inherited; `stellar-cli.md` §6; REQ-acct-subaccounts); **SEP-10** client with ephemeral auth-only keypairs as the `EphemeralAuth` signer kind (REQ-sep-sep10-client, REQ-sep-sep10-ephemeral; answers G1); **SEP-43** via the companion UI module (REQ-sep-sep43-walletkit; `freighter-walletkit.md` §9); **SEP-53** message sign/verify (inherited; reusable for SEP-10 challenges); **`stellar.toml`** resolution for identity-anchored policy (REQ-sep-stellar-toml).

Soroban auth-entry flows are first-class: `simulate` as a peer of `invoke` returning resolved tx + auth entries + footprint + resource fees (REQ-sor-simulate-first); `signAuthEntry` as a distinct operation producing a scoped `SorobanAuthorizationEntry` signature (REQ-sor-auth-entries; the primitive x402, delegation and multi-party flows depend on); **C-account as signer source** (REQ-sa-c-account-support); simulation-auth-matches-submission-auth audit (REQ-sor-simulation-audit; LaunchTube primitive lifted in-process).

SEP-24 / SEP-6 anchor flows are **OUT** in v1 (REQ-sep-out-of-scope-issuance); may ship as skills. x402-server is **OUT** (REQ-svc-x402-out-provide); consume is core.

**Machine Payments Protocol (MPP).** Option A additionally covers MPP (Tempo + Stripe, submitted to IETF Datatracker as `draft-ryan-httpauth-payment`, individual submission at `-01`) alongside x402. MPP charge mode fits naturally on the Soroban auth-entry surface described above: the client signs a `SorobanCredentialsAddress` entry for SAC `transfer`, a narrow case of `signAuthEntry`. MPP channel mode is a distinct signing primitive (off-chain cumulative commitments signed with a dedicated ed25519 commitment key) that slots in alongside the policy engine rather than on top of the auth-entry surface. Load-bearing MPP requirements: `REQ-svc-mpp-charge` (MUST), `REQ-svc-mpp-channel` (SHOULD), `REQ-svc-mpp-commitment-keys` (SHOULD), `REQ-svc-mpp-null-source-account` (MUST — the `GAAA...AWHF` sponsored-pull convention is non-obvious and easy to mishandle), `REQ-svc-mpp-challenge-validation` (SHOULD), `REQ-svc-mpp-policy-gating` (MUST — MPP has no human-in-the-loop gate in the spec; policy is the sole defence), `REQ-svc-mpp-receipt-audit` (SHOULD — correlates the challenge, on-chain reference, and resource delivery in the hash-chained audit log), `REQ-svc-mpp-x402-coexistence` (wallet chooses one protocol per 402 response), `REQ-svc-mpp-version-pinning` (SHOULD — MPP SDK is not API-stable; pinning is a release discipline requirement), `REQ-svc-mpp-mcp-transport` (SHOULD — MPP defines an MCP JSON-RPC binding via `-32042` and `_meta.org.paymentauth/*` metadata). `REQ-svc-mpp-channel-dispute` is NICE and blocked on upstream spec availability. See `research/stellar-capabilities/10-mpp.md` and `analysis/03-requirements-addendum.md` §5.10.

## 8. Delta vs. SDF + OZ collaborations (§5.4)

**Track 1 (x402-on-Stellar + x402-MCP server).** SDF's track 1 is about agents discovering paid resources and chaining paid API calls. Option A subsumes it as a narrow case of the Soroban auth-entry surface (REQ-svc-x402-consume; `_summary.md` non-wallet finding "x402 on Stellar is handled via Soroban auth entries"). Delta: (i) wallet-issued nonce binding on the simulate-commit flow (REQ-svc-x402-nonce-binding); this closes the advisory-only `confirmed: false → true` anti-pattern documented in `_tier-2/crypto/phantom-mcp-server.md` §10 (`_summary.md` cross-cutting finding 5); (ii) a local policy engine the x402 flow runs under, not a vendor-side one; (iii) the broader wallet primitive x402 plugs into.

**Track 2 (LaunchTube → OZ Relayer Channels Plugin).** Option A ships the sequence-pool primitive **in-process** (§3.5; REQ-classic-sequence-pool), removing the dependency. Delta: (i) no third-party submitter sees plaintext envelopes correlated to a bearer token (the architectural reason LaunchTube fails N2/N3, `launchtube.md` §10); (ii) no AGPL-3.0 deployment dependency if OZ Relayer's licence is confirmed; (iii) documented throughput per identity (REQ-perf-sequence-pool-throughput).

Neither track's scope matches this wallet's; positioning is additive, not oppositional. Delegates will compare each track to this work individually and both collectively.

## 9. What this option explicitly does not do

- **Does not** become a daemon. `stellar-agent-wallet mcp` is a stdio server started on demand; hosts wanting a daemon wrap the binary via their init system.
- **Does not** embed a model. The wallet is the subject of agent calls, not an agent.
- **Does not** ship `raw_sign`-style opaque-byte signing (C9 rejected; `_tier-2/crypto/privy.md` §10). Policy cannot reason about opaque bytes.
- **Does not** ship SEP-24 / SEP-6 anchor flows in v1 (REQ-sep-out-of-scope-issuance).
- **Does not** target Windows in v1 (REQ-plat-windows OUT).
- **Does not** rely on any project-operated server. No LaunchTube, no Freighter backend, no SDP. `--sign-with-lab` is blocked (`stellar-cli.md` §10).
- **Does not** make the companion UI mandatory. Headless A1 / A5 hosts run without it.
- **Does not** claim to defeat host compromise with root (`02-threat-model.md` §2.2).

## 10. Implementation cost estimate (rough)

| Component | Reuse / new | Effort (eng-months) |
|---|---|---|
| CLI shell, identity files, SEP-5, classic ops, XDR, secret format, Ledger, OS-keyring, Soroban tx assembly | Reuse `soroban_cli` crate | ~0.5 |
| Uniform JSON envelope + exit codes + error taxonomy + `--json` everywhere | New | 1-1.5 |
| MCP stdio server derived from clap metadata | New (Kraken pattern) | 1.5-2 |
| Policy engine (typed criteria + object kinds + HKDF tokens + guards + signed files + governance) | New | 3-4 |
| C-account template (Meridian Pay baseline + OZ accounts + `_auth_contexts` inspection + policy-initiated exec + sessions + passkey) | Partial reuse; new policy extensions | 3-4 |
| In-process sequence pool | Pattern reused, new code | 2 |
| Approval channel + wallet-issued nonces + novel-field highlighting | New | 1.5-2 |
| Companion UI + SEP-43 + Wallet Kit + WalletConnect v2 w/ `signAuthEntry` / `signMessage` | Partial reuse; new app | 3-4 |
| Skills: external-binary + sandbox + pinned install + audit registry + manifest + first-invoke gate | Pattern reused; sandbox new | 2-3 |
| Hash-chained audit log + receipts store | New | 1-1.5 |
| Packaging: binaries, reproducible builds, signed releases | Standard | 1 |
| x402 consume + receipt correlation | Narrow Soroban auth-entry use | 1 |
| MPP charge + channel modes + MCP binding (atop x402 scaffolding) | Incremental over x402 path | 1-1.5 |
| SEP-10 client + ephemeral keys | Straightforward | 0.5-1 |
| Test harness (testnet integration, policy / amount fuzzing, audit-log tamper, schema) | Ongoing | 2-3 front-loaded |

**Rough total**: ~23-30 engineering-months for v1. T-shirt: **L**.

Biggest cost-reducer: the `soroban_cli` crate. Biggest cost-contributor: the policy engine plus its on-chain counterpart. This is where option A adds capability over existing N1/N2/N3-passing references (Safe on EVM; `stellar-cli` as non-agent tooling).

## 11. Primary risks

- **`soroban_cli` library-crate stability.** `stellar-cli.md` §11 flags that `pub commands::*` stability across 26.x is undocumented and crates.io consumption vs. workspace-only use is untested. *Mitigation:* pin the version, run against a local mirror, upstream a stability contract.
- **`--secure-store` feature availability.** The OS-keyring signer is behind the `additional-libs` feature and may not ship in all binstall / brew channels (`stellar-cli.md` §11). *Mitigation:* wallet releases build it in regardless.
- **Companion UI adoption lag.** If A4 users arrive before v1 of the companion, they have no wallet-owned approval channel and T9 regresses. *Mitigation:* ship a minimal desktop companion in the v1 cycle; publish WalletConnect v2 early so existing mobile wallets serve as approval transport. Scope is materially smaller than it might appear because two reference implementations already cover the smart-account half of the companion — kalepail SAK for web and Soneso KMP `OZSmartAccountKit` for Android / iOS / macOS / web with a 7-screen demo including Reown WalletConnect v2 (see `research/external/stellar-ecosystem/smart-account-kit.md`). The wallet's companion is a composition and extension of these SDKs plus the wallet-owned approval-channel logic; `REQ-sa-companion-sdk-composition` (SHOULD) names this explicitly.
- **Skill-ecosystem supply chain.** Extensibility is a T8 vector unless the attestation-registry story is enforced from day one (`safe.md` §10, `_tier-2/crypto/metamask-snaps.md` §10). *Mitigation:* ship signed-install + audit-gate before arbitrary-publisher skills; start first-party-only.
- **OZ `stellar-contracts` evolution.** The C-account template builds on OZ's accounts pattern (`meridian-pay.md` §13); OZ's API is young. *Mitigation:* pin an audited OZ version; contribute policy-initiated-exec upstream.
- **LaunchTube migration timing.** SDF's track-2 migration lands during the expected v1 development window. *Mitigation:* sequence-pool is v1 scope, not deferred.

## 12. Honest self-assessment

Where option A is **weaker** than a fully agent-native design (option B):

- **Non-MCP agent ergonomics.** The canonical surface is the CLI; MCP is derived, a library embed path exists. An agent runtime speaking neither MCP nor a Rust FFI — a Python or Go agent on a custom tool protocol — shells out and parses the envelope. Second-class compared to a design whose primary surface is a typed-tool server exposed over several protocols at once.
- **Schema-evolution discipline.** The MCP schema is derived from clap metadata, so a careless CLI rename propagates as a breaking change across every MCP consumer. An agent-native design forces discipline structurally; option A enforces it through process.
- **Companion UX cohesion.** Companion-downstream-of-CLI is honest about provenance but means the companion and the CLI evolve as two sub-projects. An agent-native design that makes the approval-channel IPC first-class gets cohesion for free.
- **Policy-as-protocol.** The policy engine is a core library called by both transports. Exposing policy evaluation itself as a callable surface to other wallets is additional work.

Where option A is **stronger** than option B:

- **Lowest implementation cost.** `soroban_cli` and the external-binary plugin exist today; the foundation lands in days. Option B starts from a blank MCP server and builds the command surface alongside the tool surface.
- **Incumbent discovery.** `stellar <subcommand>` is muscle memory; option A inherits it.
- **Human-path strength.** CLI with table output and dry-run is A7's native environment; humans are the last recovery path under every actor.
- **Stellar-ecosystem positioning.** "Extend the CLI you already maintain, fill in what it is missing, ship the additions as a plugin." Concrete and easy to evaluate.

Neither option strawmans the other. Option A acknowledges MCP exists, matters, and ships in v1; option B acknowledges a CLI falls out and matters for humans. The decision is about **which surface is the source of truth**, not which exists.

## 13. Cross-references to the companion option

- `analysis/07-option-b-agent-native.md` — peer draft. Where option A derives MCP from CLI, option B derives CLI from MCP.
- `analysis/08-option-c-layered.md` — honest hybrid. Option A is already a hybrid with a named center of gravity; option C asks whether refusing to name one is viable.
- `analysis/09-decision-matrix.md` — axis-weighted comparison.
- `analysis/10-recommendation.md` — recommendation document.

### Requirement citations

REQ-IDs cited inline are drawn from every section of `03-requirements.md` and the stream-2 adds in `03-requirements-addendum.md`. Coverage by area: account + key management (REQ-acct-*), classic ops (REQ-classic-*), Soroban (REQ-sor-*), smart-account + delegation (REQ-sa-*), SEP flows (REQ-sep-*), agentic-payment services — x402 (REQ-svc-x402-*) and MPP (REQ-svc-mpp-*), observation (REQ-obs-*), security + policy (REQ-sec-*), UX + MCP (REQ-ux-*), audit (REQ-audit-*), performance (REQ-perf-*), extensibility (REQ-ext-*), packaging + platform + config + telemetry (REQ-pkg-*, REQ-plat-*, REQ-cfg-*, REQ-tel-*). See `03-requirements.md` and `03-requirements-addendum.md` for the authoritative IDs; this document cites IDs at the point of use.

### Research citations

Sources cited inline; paths are under `research/external/` unless tier-2, in which case `research/external/_tier-2/`.

Stellar-ecosystem: `stellar-cli.md` §5-§11; `meridian-pay.md` §4, §9, §10, §13; `freighter-walletkit.md` §9, §10; `stellar-mcp-server.md` §9, §10. Crypto tier-1: `kraken-cli.md` §9; `moonpay-agents-ows.md` §9; `safe.md` §9; `coinbase-agentic.md` §9; `turnkey.md` §9; `trust-wallet-agent-kit.md` §9. Non-crypto: `gh-cli.md` §9, §10; `stripe-cli.md` §9; `aws-cli.md` §9, §10. Tier-2 crypto: `fireblocks.md` §9; `metamask-snaps.md` §9, §10; `privy.md` §9, §10; `tether-wdk.md` §9, §10; `phantom-mcp-server.md` §9, §10. Tier-2 Stellar: `launchtube.md` §9, §10. Rollup: `_summary.md` cross-cutting findings 1, 2, 4, 5, 6, 7.
