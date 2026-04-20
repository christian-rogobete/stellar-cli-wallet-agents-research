# Smart Account Kit (SAK + KMP OZSmartAccountKit)

**Status:** complete
**Researcher:** supporting-reviewer research pass, 2026-04-20
**Last updated:** 2026-04-20
**ID:** smart-account-kit
**Category:** crypto-stellar

**Scope note.** Two SDKs are in active development against OpenZeppelin's `stellar-accounts` v0.7.1 on-chain contracts: the TypeScript **Smart Account Kit (SAK)** by kalepail at `github.com/kalepail/smart-account-kit` and the Kotlin-Multiplatform **OZSmartAccountKit** maintained by Soneso as part of `kmp-stellar-sdk`. Both target the same on-chain contract, expose the same manager model, and produce interoperable contract addresses (subject to byte-identical credential-ID encoding across passkey-provider outputs — §11). This file covers both as "two implementations of one pattern" and calls out platform-specific differences where they matter.

Commit pins: SAK at `e731a3d` (version 0.3.0); KMP at `c7bfedb`. Short names `smart-account-kit` and `kmp-stellar-sdk` per `SOURCES.md`.

---

## 1. What it is

Two client SDKs for deploying and managing OpenZeppelin smart-account contracts on Stellar / Soroban. SAK targets the browser (TypeScript, IndexedDB-backed credential storage, WebAuthn via the browser Credential API). KMP `OZSmartAccountKit` targets Android, iOS, macOS, and web (Compose Multiplatform + platform-specific WebAuthn providers). Both handle passkey registration/authentication ceremonies, assemble Soroban `AuthPayload` with `context_rule_ids`, sign auth entries, and submit transactions either directly via Soroban RPC or through an optional fee-sponsoring relayer. Neither is a wallet in itself — both are libraries a wallet or demo app uses to drive an on-chain smart account.

## 2. Agent interface

Both SDKs expose an `OZSmartAccountKit` (KMP) / `SmartAccountKit` (SAK) facade with identical sub-manager split:

- `walletOperations` / (SAK: top-level) — `createWallet`, `connectWallet` (with silent-restore, prompt, fresh, known-credentials modes), `disconnect`, `deployPendingCredential`.
- `transactionOperations` / `signAndSubmit` / `transfer` / `executeAndSubmit` — signs Soroban auth entries and submits. `executeAndSubmit` routes through the account's `ExecutionEntryPoint::execute`, which is the only way a client can reach arbitrary target contracts via the smart account as authoriser.
- `signerManager` — `addPasskey`, `addDelegated`, `addExternal` (Ed25519 or verifier-wrapped), `removeSigner`.
- `contextRuleManager` / `rules` — `add`, `get`, `list`, `getAll`, `remove`, `updateName`, `updateExpiration`.
- `policyManager` / `policies` — `add` (with variants `addSimpleThreshold`, `addSpendingLimit`, `addWeightedThreshold` for KMP), `remove`.
- `credentialManager` — storage-side credential lifecycle.
- `multiSignerManager` — `getAvailableSigners`, `needsMultiSigner`, `buildSelectedSigners`, multi-signer `transfer` / `operation`.
- `externalSigners` (SAK) / `externalWallet` (KMP) — adapter for G-address signers (keypair in memory or from a Stellar Wallets Kit / WalletConnect adapter).
- `events` — `walletConnected`, `walletDisconnected`, `credentialCreated`, `credentialDeleted`, `sessionExpired`, `transactionSigned`, `transactionSubmitted`.

Minimum sequence for a signed transfer (both SDKs, pseudocode):

```
kit = create(config)  // rpcUrl, networkPassphrase, accountWasmHash, webauthnVerifierAddress
connection = kit.connectWallet()  // silent or WebAuthn ceremony
if (connection == null) kit.createWallet(userName, autoSubmit=true)
result = kit.transactionOperations.transfer(tokenContract, recipient, amount)
```

Signing flow depends on which signer type governs the active context rule:

- **Passkey signer (`Signer::External` via WebAuthn verifier).** Each signing action triggers a WebAuthn ceremony on the platform's OS-level biometric UI (Face ID / fingerprint / security key on the *user's* device). **This path requires a human at the device.** It is the right flow for the companion UI approving high-value actions, not the default for an unattended agent.
- **Delegated G-account signer (`Signer::Delegated(Address)`).** The signer is a classic G-account keypair; signing is a standard ed25519 signature over the Soroban auth-entry payload. No biometric prompt. The keypair can live in an agent's keystore, a CI secret, a hardware wallet driven programmatically, or an external wallet adapter. **This is the agent-path signer.**
- **External ed25519 signer (`Signer::External` via the ed25519 verifier contract).** Same ed25519 signature mechanics as delegated but through a verifier abstraction; address-less pubkeys reduce the T1 exfil value (see `analysis/02-threat-model.md` T1 defences).

An agent flow (actors A1 daemon, A2 orchestrator, A3 service-consumer, A5 CI/CD) **MUST NOT rely on the passkey path** — by construction there is no human to prompt. The agent uses a delegated or external signer governed by a dedicated context rule with tight policy; the passkey is reserved for the user's companion-UI actions.

## 3. Custody model

Custody tier depends on signer type.

**Passkey signers (user's device; companion-UI path).** Keys are WebAuthn passkeys held by the OS-level authenticator (Android Credential Manager, iOS/macOS AuthenticationServices, browser-bound passkey platform). The SDK never sees or exports the private key; it receives a `WebAuthnSigData` attestation/assertion object signed by the authenticator and verified on-chain by the deployed WebAuthn verifier contract (secp256r1). There is no key backup path inside the SDK — authenticator sync (iCloud Keychain, Google Password Manager) is the recovery boundary. Because this path requires a biometric prompt, it is available to a human user but not to an unattended agent.

**Delegated and external signers (agent-path and hardware paths).** `Signer::Delegated(Address)` and `Signer::External(Address, Bytes)` support any signer whose private material is a classic ed25519 keypair: agent-keystore entries, CI / OIDC-issued short-lived credentials, hardware wallets driven programmatically, in-memory seed for tests, or external wallet adapters (Stellar Wallets Kit browser extension, WalletConnect v2 / Reown to a mobile wallet). These paths sign Soroban auth entries without a biometric prompt and are therefore the wallet-host-side path available to agent actors.

No vendor-side custody: neither SDK has a server component in the signing path. The optional indexer (§4) and optional relayer are both non-custodial — the indexer reads the ledger and exposes reverse lookups; the relayer fee-bumps a client-signed envelope but cannot modify it.

## 4. Policy and approval model

Policy lives **on-chain** in the OZ `stellar-accounts` contract. The client SDKs expose three built-in policy types matching OZ's shipped policies plus an `addPolicy(address, installParams)` escape hatch for custom wasm:

- **Simple-threshold** — M-of-N multisig gate, M applied uniformly to rule's signer set.
- **Weighted-threshold** — per-signer weight + threshold.
- **Spending-limit** — rolling-window spending cap per token, configured as `(token_contract, limit_stroops, period_ledgers)`.

Each context rule supports **up to 5 policies and up to 15 signers** (OZ-contract hard caps, validated client-side before submission — see `research/stellar-capabilities/04-smart-accounts.md` §2). Rules themselves are unbounded per account.

Approval is transaction-time only: every signature requires a fresh biometric prompt (or fresh external-wallet ceremony). The SDK does **not** implement client-side policy (spending caps outside the contract, counterparty allowlists that are not stored on-chain as policies, etc.) — policy is authoritative on-chain. A wallet built on top of either SDK would layer its own local-policy engine before calling the SDK (this is the pattern our wallet adopts under `REQ-sec-local-policy-engine`).

Policy mutations (`kit.policies.add`, `kit.policies.remove`, `kit.signers.*`, `kit.rules.*`) are themselves signed transactions gated by the matching context rule — which is the account itself via `ExecutionEntryPoint`. Who can change them is therefore whichever rule the account grants self-authorization to.

## 5. Delegation model

Delegation is expressed in OZ's native form: one `ContextRule` per sub-agent, each with its own signer set, policies, and optional `valid_until` ledger bound. Both SDKs expose this shape directly:

```
rules.add(contextType, name, signers, policies)
rules.updateExpiration(id, absoluteLedger)
rules.remove(id)
```

Authority is **scoped by context type**: `Default` matches every auth context; `CallContract(addr)` restricts to calls at a specific contract; `CreateContract(wasmHash)` restricts to contract deployments.

Revocation is `rules.remove(id)` — takes effect at the next `__check_auth` evaluation against that rule. An already-signed envelope referencing that rule ID can still succeed if submission races the revocation ledger (see `02-threat-model.md` T12 for the narrower threshold-brick case). `valid_until` bounds the exposure window for newly-constructed signatures without requiring explicit revocation.

Neither SDK exposes a policy-initiated-execution path (OZ v0.7.1 does not provide one — see `04-smart-accounts.md` §7). Recurring payments or top-ups would be scheduled by the wallet and authorised under a scoped rule; the SDK does not ship a scheduler.

## 6. Relevant protocol coverage

**Stellar / Soroban.** Custom-account `__check_auth`; `SorobanAuthorizationEntry` construction and signing; smart-account contract deployment (WASM hash + constructor args); `ExecutionEntryPoint::execute` for arbitrary self-authorised calls; secp256r1 / WebAuthn signer verification via on-chain verifier contract; Ed25519 signer verification via a separate verifier contract. Both SDKs compute and sign the OZ v0.7 auth digest `sha256(signature_payload || context_rule_ids.to_xdr())` — the preimage that binds `context_rule_ids` into the signature and closes the rule-ID-downgrade vector (`analysis/02-threat-model.md` T11).

**Classic Stellar surfaces used implicitly.** G-address deployer keypair (the source account for the deploy transaction); optional relayer fee-bump path; `FeeBumpTransaction` submission when a relayer is configured.

**External interop.** KMP demo ships a **Reown (WalletConnect v2) client** for Android and iOS to connect Freighter Mobile as a delegated signer; browser-based SAK demo integrates Freighter via its browser extension; both support `ExternalWalletAdapter` / `ExternalSigner` plug-in points for any Stellar Wallets Kit module.

## 7. Licence and adoption signals

- **SAK licence:** Apache-2.0 (verbatim `LICENSE` file in the repo; `package.json` `"license"` field agrees). Source available at `github.com/kalepail/smart-account-kit`. Version 0.3.0 at commit `e731a3d` (2026-04 timeframe). Pre-1.0; shape expected to iterate alongside OZ `stellar-accounts`.
- **KMP OZSmartAccountKit licence:** Apache-2.0 (verbatim `LICENSE` file in `kmp-stellar-sdk`; repo README states "Licensed under the Apache License, Version 2.0"). Source available at `github.com/Soneso/kmp-stellar-sdk`. Part of the maintained SDK line (Flutter, iOS, PHP, KMP) listed in `SOURCES.md`.
- **Last meaningful commit:** both under active development as of 2026-04. SAK at `e731a3d`; KMP at `c7bfedb`.
- **Known adopters (demo-stage).** SAK drives the kalepail passkey-kit demo flows and the OZ stellar-contracts demo app; KMP ships as part of the Soneso SDK with its own 4-platform demo. Production deployments of either in third-party apps are nascent given OZ `stellar-accounts` is itself pre-1.0.
- **Commercial backing:** kalepail is independent (same developer behind passkey-kit and LaunchTube); Soneso maintains the KMP SDK as part of its Stellar SDK product line.
- **Ecosystem traction:** the two SDKs produce **interoperable contract addresses** from the same credential + deployer inputs (both use `SHA256("openzeppelin-smart-account-kit")` as the default deployer seed — see §9 item "deterministic-address derivation"), which means a wallet created through one can be reopened through the other. This is a strong interop property and a good sign for ecosystem health.

## 8. Non-negotiable check

Against `analysis/00-context.md` §4:

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | Keys are OS-level passkeys or local keypairs; neither SDK stores private key material. |
| N2 | Autonomous (no project-operated backend) | **partial** | Both SDKs are self-contained for on-chain operations. **Rule enumeration on a known account** is handled differently by each: **KMP** enumerates active rules directly from the chain via `OZContextRuleManager.getAllContextRules()` — scans `[0, get_context_rules_count())` and filters deleted rule IDs client-side; no indexer needed. **SAK** defaults `rules.list()` / `rules.getAll()` to the hosted indexer. **Credential-to-contract reverse lookup** ("which contract does this passkey credential control?") and **multi-signer discovery** default to the hosted indexer in both SDKs when the caller doesn't already know the contract address; the alternative is deterministic-address derivation from the default deployer seed (see §9). A wallet that ships with the default indexer URL unchanged introduces a third-party liveness dependency — if the indexer is unreachable, the affected discovery features degrade. Defaults point at `smart-account-indexer.sdf-ecosystem.workers.dev` (testnet) / `smart-account-indexer-mainnet.sdf-ecosystem.workers.dev` (mainnet), Cloudflare Workers on the `sdf-ecosystem.workers.dev` subdomain. A production wallet either self-hosts, relies on deterministic-address derivation, maintains a local index, or accepts the dependency. |
| N3 | No central server for keys/policy/history | **yes** | The indexer reads only public on-chain event data (smart-account signer and context-rule events emitted by OZ `stellar-accounts` — `research/stellar-capabilities/04-smart-accounts.md` §7). It holds no keys, no policy state, no transaction history beyond what is already public on-chain, and no user-provided data. Residual concern is a **query-pattern side channel** — an indexer operator can observe which credential IDs or account addresses a wallet queries over time, which is a traffic-analysis concern rather than a data-exposure concern. Comparable in kind to querying any public explorer API; materially weaker than LaunchTube's plaintext-envelope-with-bearer-token pattern. Does not fail N3 on data-exposure grounds. |
| N4 | Open source, permissive licence | yes (SAK Apache-2.0; KMP Apache-2.0) | Both sources are published under Apache-2.0. |
| N5 | JSON-default output | n/a | These are library SDKs, not CLIs; output shape is determined by the consuming application. |
| N6 | Testnet/mainnet parity | yes | Configuration parameterised by `networkPassphrase` and `rpcUrl`; no testnet-only assumptions in either SDK core. |

**Conclusion.** Both SDKs are usable as *design references* and as *direct library dependencies*. The N2 concern is a liveness dependency for the features that default to the indexer (most relevant: credential-to-contract reverse lookup); a wallet that wants to avoid it either uses KMP's native on-chain scan for rule enumeration, relies on deterministic-address derivation for contract discovery, self-hosts the indexer, or maintains a local index. N3 is not at risk — the indexer reads only public on-chain data — but a wallet that cares about query-pattern privacy should still self-host.

## 9. What to adopt

- **OZ-native delegation model exposed as sub-managers.** Both SDKs converge on the same CLI-like shape: one manager per concern (`signers`, `rules`, `policies`, `multiSigners`, `credentials`, `events`, `externalSigners`). Our CLI command tree should mirror this for ecosystem-familiarity. *Surfaces `REQ-sa-manager-surface` — new.*
- **Deterministic-address derivation.** `SHA256("openzeppelin-smart-account-kit")` as the default deployer seed across SAK and KMP produces identical contract addresses from identical credential IDs. A wallet can discover the user's contract address **without** an indexer by re-deriving it locally, as long as the deployer seed is known. Our wallet should ship the same derivation helper and document the seed as a convention. *Surfaces `REQ-sa-deterministic-address-derivation` — new.*
- **Wrapper-versus-raw discipline.** SAK explicitly: "if the SDK adds orchestration, session handling, signer resolution, or submission logic, keep the wrapper. If the method is just a thin contract call, use `kit.wallet` directly." Methods kept raw: `upgrade`, `batch_add_signer`, `get_signer_id`, `get_policy_id`, `get_context_rules_count`. Our wallet should adopt the same split — polish common flows, expose raw contract calls as an escape hatch rather than reimplementing them. *Surfaces `REQ-sa-wrapper-raw-discipline` — new.*
- **Context-rule type builders.** `createDefaultContext`, `createCallContractContext(address)`, `createCreateContractContext(wasm_hash)` — named constructors matching OZ's `ContextRuleType` enum variants. CLI UX should expose these as positional subcommands (`wallet rules add --context default`, `--context call-contract --address C...`, `--context create-contract --wasm-hash 0x...`). *Cited in `REQ-sa-context-rule-caps` and `REQ-sa-manager-surface`.*
- **Policy builder primitives.** `createThresholdParams(m)`, `createWeightedThresholdParams(weights, threshold)`, `createSpendingLimitParams(token, limit, period_ledgers)`. Constants like `LEDGERS_PER_HOUR`, `LEDGERS_PER_DAY`, `LEDGERS_PER_WEEK` shipped as named utilities. The CLI should provide these as named durations rather than forcing users to compute ledger counts.
- **Two-phase connect pattern.** `connectWallet()` silently restores from stored session; `connectWallet({prompt: true})` triggers WebAuthn; `connectWallet({fresh: true})` bypasses the session; `connectWallet({credentialId, contractId})` connects directly with known state. A CLI wallet can mirror this for attended dev workflows: `wallet connect` tries silent, `wallet connect --prompt` forces a fresh passkey ceremony.
- **Multi-signer flow coordination.** `multiSigners.buildSelectedSigners()` + `multiSigners.operation()` as the named pattern for multi-party authorisation. Our wallet's multisig UX should reuse this vocabulary.
- **Event system for wallet lifecycle.** `walletConnected`, `credentialCreated`, `transactionSigned`, `transactionSubmitted`, `sessionExpired` as first-class events. Maps naturally to MCP notifications in an agent-wallet setting.
- **Companion UI reference implementations.** Both SDKs ship demo apps, with different platform reach. **SAK's demo** is a Vite + React single-page app (`demo/` folder) — **web-only**; it covers wallet create, connect, context-rule management with signers and policies, multi-signer transfer, and Freighter-browser-extension integration as the external-wallet path. **KMP's demo** (`smart-account-demo/`) covers **four platforms — Android, iOS, macOS, and Web** — with 7 screens (dashboard, wallet creation, connection, transfer, context-rules list, rule builder, account signers) and integrates Reown WalletConnect v2 for connecting Freighter Mobile as a delegated signer on Android and iOS. Decomposition for a production companion: **SAK is a candidate for a web-only companion surface; KMP is a candidate for native mobile, native desktop, and a shared code path on web.** They can coexist (same contract addresses, same signer wire format), so a wallet may ship a web SPA derived from SAK and native apps derived from KMP without forking the smart-account layer. *Surfaces `REQ-sa-companion-sdk-composition` — new, SHOULD.*

## 10. What to avoid

- **Default-indexer URL shipped unchanged in production.** The hardcoded `sdf-ecosystem.workers.dev` URLs are convenient for demos but carry a third-party liveness dependency if a production wallet ships with them as-is. The data served is public on-chain (so there is no data-exposure concern; see §8 N3 row), but indexer downtime degrades discovery features, and the indexer operator sees which credential IDs the wallet queries over time (traffic-analysis / query-pattern side channel). A production wallet either sets `indexerUrl` to its own instance, unsets it and relies on alternative paths (deterministic-address derivation for contract discovery, on-chain scan for rule enumeration on a known account), or accepts the dependency with eyes open.
- **Active-rule enumeration path differs between SDKs.** The on-chain contract exposes `get_context_rule(id)` and `get_context_rules_count()` but no active-rule-ID iterator after deletions. **KMP's `OZContextRuleManager.getAllContextRules()` scans `[0, count)` on-chain and filters deleted entries client-side** — no indexer needed for rule enumeration on a known account. **SAK's `rules.list()` / `rules.getAll()` defaults to the hosted indexer.** A self-custodial wallet that uses SAK for rule enumeration either re-implements the scan pattern atop `kit.wallet.get_context_rule(id)` + `kit.wallet.get_context_rules_count()` or maintains a local index updated on every rule mutation it signs. Reverse lookups (credential → contract address) are a separate question and are indexer-backed in both SDKs unless the caller uses deterministic-address derivation. *Surfaces `REQ-sa-active-rule-enumeration` — new; the MUST applies to the wallet's own implementation, not to the choice of SDK.*
- **Default deployer as the real deployment account in production.** The `SHA256("openzeppelin-smart-account-kit")` default is documented as *"publicly derivable ... intended to be used with a relayer that sponsors transaction fees, or funded externally."* Using it in production leaves contract deployments attributed to a shared, well-known keypair. The intent is ecosystem-interop — a wallet implementation should still use a branded deployer for production deployments (KMP README §"Custom Deployers"), reverting to the well-known seed only for interop verification.
- **Browser-bound storage adapters in a CLI context.** SAK's `IndexedDBStorage` / `LocalStorageAdapter` are browser-specific; a CLI wallet cannot reuse them. Our wallet would supply its own `StorageAdapter` implementation backed by the OS keyring. KMP is more broadly applicable here because its storage interface is deliberately abstract.
- **Session auto-restore inside a CLI** without additional guards. `connectWallet()` silent restore is right for a long-lived companion app. A CLI process short-lived by nature (one command, exit) should either pass explicit `credentialId` + `contractId` per invocation or gate session restore behind an unlock step consistent with `REQ-sec-short-unlock`.

## 11. Open questions

- **Indexer protocol stability.** Both SDKs index against the `smart-account-indexer` Cloudflare Worker; the indexer protocol (request/response shape, rate limits, retention policy) is not formally versioned. A wallet running its own indexer needs a protocol spec or accepts SAK/KMP as the de facto implementation reference.
- **Deployment-interop edge cases.** Default deployer produces identical addresses across SDKs only if the credential ID representation is identical. Both SDKs hash raw credential ID bytes; verify that the byte-level encoding of the credential ID is the same in both (base64url vs raw bytes, trailing padding) so interop holds under all passkey-provider outputs.

## 12. Sources

- SAK README: [`smart-account-kit/README.md`](https://github.com/kalepail/smart-account-kit/blob/main/README.md) @ `e731a3d` — accessed 2026-04-20 — package overview, API surface, environment configuration, build instructions.
- SAK source: [`smart-account-kit/src/`](https://github.com/kalepail/smart-account-kit/tree/main/src) @ `e731a3d` — `kit.ts` (default deployer seed), `indexer.ts` (default URLs, `sdf-ecosystem.workers.dev`), `managers/` (sub-manager implementations). Commit pinned.
- SAK indexer: [`smart-account-kit/indexer/README.md`](https://github.com/kalepail/smart-account-kit/blob/main/indexer/README.md) @ `e731a3d` — Goldsky pipeline + PostgreSQL + Cloudflare Worker architecture; confirms indexer is a project-operated service.
- KMP OZSmartAccountKit README: [`kmp-stellar-sdk/docs/smart-accounts/README.md`](https://github.com/Soneso/kmp-stellar-sdk/blob/main/docs/smart-accounts/README.md) @ `c7bfedb` — accessed 2026-04-20 — architecture diagram, config reference, manager breakdown, deterministic address derivation.
- KMP source: [`kmp-stellar-sdk/stellar-sdk/src/commonMain/kotlin/com/soneso/stellar/sdk/smartaccount/oz/`](https://github.com/Soneso/kmp-stellar-sdk/tree/main/stellar-sdk/src/commonMain/kotlin/com/soneso/stellar/sdk/smartaccount/oz) @ `c7bfedb` — `OZSmartAccountConfig.kt` (default deployer seed; default indexer URLs per network passphrase), `OZIndexerClient.kt`.
- KMP demo: [`kmp-stellar-sdk/smart-account-demo/README.md`](https://github.com/Soneso/kmp-stellar-sdk/blob/main/smart-account-demo/README.md) @ `c7bfedb` — Android / iOS / macOS / Web targets, 7 screens, Reown WalletConnect v2 integration, Freighter Mobile as external delegated signer.
- KMP WebAuthn platform integration: [`kmp-stellar-sdk/docs/smart-accounts/webauthn-{android,ios,macos,web}.md`](https://github.com/Soneso/kmp-stellar-sdk/tree/main/docs/smart-accounts) @ `c7bfedb` — per-platform passkey configuration.
- OpenZeppelin contracts: [`stellar-contracts/packages/accounts/README.md`](https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/README.md) @ `3f81125` — the on-chain contracts both SDKs drive.

## 13. Cross-links

- **Requirements surfaced (for `03-requirements-addendum.md` §5.4):**
  - `REQ-sa-manager-surface` (SHOULD) — CLI command tree mirrors the manager split for ecosystem-consistent UX.
  - `REQ-sa-deterministic-address-derivation` (SHOULD) — wallet ships the `SHA256("openzeppelin-smart-account-kit")` derivation helper so contracts can be discovered without an indexer.
  - `REQ-sa-active-rule-enumeration` (MUST) — wallet enumerates active rules without a hosted indexer dependency (local index maintained on every signed mutation, or incremental id-scan).
  - `REQ-sa-wrapper-raw-discipline` (SHOULD) — wallet CLI wraps orchestration / session / signer-resolution / submission logic and exposes raw contract calls as an explicit escape hatch.
  - `REQ-sa-companion-sdk-composition` (SHOULD) — companion UI composed from SAK (web) and/or KMP `OZSmartAccountKit` (native multiplatform) rather than bespoke.
- **Other candidates compared to:**
  - `research/external/stellar-ecosystem/meridian-pay.md` — single-file ~100-LOC smart account, policy-less; these SDKs are the policy-aware successor.
  - `research/external/stellar-ecosystem/freighter-walletkit.md` — the external-signer adapter both SDK demos plug into for G-address delegation.
  - `research/external/_tier-2/stellar-ecosystem/launchtube.md` — the submitter role the optional relayer plays; SDKs' relayer abstraction is the migration path.
  - `research/external/crypto/safe.md` — the non-Stellar module-attachment-as-delegation reference. OZ-native rules-plus-policies + these SDKs are the Stellar-side answer.
- **Analysis files citing this entry:**
  - `research/stellar-capabilities/04-smart-accounts.md` §2, §4, §9 — on-chain contract + client SDK composition + cross-link list.
  - `analysis/06-option-a-cli-first.md` §3.6 and §11 — companion UI plus primary-risks downgrade.
  - `analysis/07-option-b-agent-native.md` §3.7 — companion UI.
  - `analysis/02-threat-model.md` T8 — hosted-indexer dependency sub-concern.
  - `analysis/03-requirements-addendum.md` §5.4 — candidate requirements.
