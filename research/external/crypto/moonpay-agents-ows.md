# MoonPay Agents + Open Wallet Standard

**Status:** complete
**Researcher:** research-agent
**Last updated:** 2026-04-18
**ID:** moonpay-agents-ows
**Category:** crypto-agent

---

## 1. What it is

**Open Wallet Standard (OWS)** is an MIT-licensed specification plus Rust reference implementation (`open-wallet-standard/core`, v1.3.1, 317 stars) for local, policy-gated signing. It defines seven sub-specs: storage format, signing interface, policy engine, agent-access layer, key isolation, wallet lifecycle, and supported chains [1, 4]. **MoonPay Agents** (launched Feb 2026) is the commercial product that instantiates OWS via the `@moonpay/cli` package, exposing 54 tools across 17 "skills" over CLI, local MCP server (`mp mcp`), and REST [8, 10]. OWS was open-sourced 23 March 2026 with 15+ contributors: PayPal, OKX, Ripple, Tron, TON Foundation, Solana Foundation, Ethereum Foundation, Base, Polygon, Sui, Filecoin Foundation, LayerZero, Dflow, Uniblock, Virtuals, Arbitrum, Dynamic, Allium, Simmer.Markets, Circle [2].

## 2. Agent interface

Three equal-rank surfaces: CLI (`ows` / `mp`), Node.js+Python SDK, and MCP server (stdio, in the MoonPay product layer, not the spec itself) [1, 8]. The normative Agent Access Layer (doc 04) defines three implementation profiles — in-process, subprocess, local-service — and mandates that all must agree on wallet lookup, policy order, canonical errors, and audit-log side effects [5]. MCP is not enumerated as a normative profile; it is a product instantiation of Profile C. Minimum signed-action sequence for an unattended caller:

```
ows wallet create --name "agent-treasury"
ows policy create --file policy.json           # declarative + optional executable
ows key create --name "agent" --wallet agent-treasury --policy <id>
# → prints ows_key_... once (token = capability + HKDF-SHA256 decryption input)
OWS_KEY=ows_key_... ows sign tx --wallet agent-treasury --chain eip155:8453 --tx 0x02...
```

Tokens are capability bearers: the wallet secret is re-encrypted under a token-derived key, so policy evaluation must pass before the agent's own token can decrypt anything [3].

## 3. Custody model

Single local vault at `~/.ows/` (700/600 permissions) holding Ethereum Keystore v3-extended files: AES-256-GCM ciphertext over BIP-39 entropy or raw private keys, with scrypt (min N=2^16) for owner passphrases and HKDF-SHA256 for API-token key derivation [7]. At sign time: (1) policy evaluates against the token's scope, (2) secret decrypts into `mlock()`-ed protected memory, (3) signing key is derived, (4) payload is signed, (5) buffers are zeroized. Cached key material has a 30-second max TTL with LRU eviction [6]. The API never returns raw keys except via an explicit `wallet export` with visible warning [7]. Ledger hardware signing is **not in the normative OWS spec**; it is a MoonPay Agents product feature announced March 2026 [2, 10]. The "subprocess enclave" mode (key never touches the parent agent process) is documented as a future direction, not current [6].

## 4. Policy and approval model

Policies attach to API keys (not wallets). Evaluation is AND across all declarative rules, then an optional executable (5-second subprocess timeout, fail-closed on timeout/non-zero/malformed output) [3]. **The spec defines exactly three declarative rule types**: `allowed_chains` (CAIP-2 allowlist), `expires_at` (ISO-8601), `allowed_typed_data_contracts` (EIP-712 contract allowlist) [3]. **Per-transaction amount caps, cumulative spend limits, destination/counterparty allowlists, and rate limits are explicitly NOT declarative rules** — the spec directs implementers to use the `executable` plugin path for these [3]. That executable is a user-installed script invoked per-request; it can call RPCs or external services. All policy decisions land in an append-only JSONL audit log [7]. Owner-mode operations bypass policy entirely [3]. No on-chain policy; enforcement is purely client-side.

## 5. Delegation model

Delegation is expressed as **scoped API tokens with attached policies**, not key shares. Revocation is `ows key revoke` and takes effect on the next sign call (no queued-tx grace window). Tokens double as decryption capability (token-derived HKDF re-wrap of the vault secret), so revoking removes decryption authority, not just authentication [3]. There is no on-chain delegation primitive — no multisig, no account-abstraction hooks, no Soroban-auth-equivalent. A compromised host with an unrevoked token retains full token-scope access.

## 6. Relevant protocol coverage

Ten chain families at v1.3.1: EVM, Solana, Bitcoin, Cosmos, Tron, TON, Sui, Spark, Filecoin, XRPL [1]. One BIP-39 mnemonic derives all families via per-chain BIP-44 coin types. **Stellar is absent from the supported-chains table** [1, 2]. Chain addition requires (per doc 07): CAIP-2 identifier, derivation path, address encoding, signing-behaviour spec — no core changes needed, pluggable by design [9]. x402 support is first-class: `ows pay request` auto-signs EIP-3009 `TransferWithAuthorization` for USDC on 402 responses; `ows pay discover` locates providers [1]. Adapters ship for viem, @solana/web3.js, and Tether WDK [1]. Signing interface: `sign`, `signAndSend`, `signMessage`, `signTypedData`, `signHash`, `signAuthorization` (EIP-7702). **No nonce management** — left to caller [4].

## 7. Licence and adoption signals

- **Licence:** MIT [1, 2]
- **Source available:** yes (`github.com/open-wallet-standard/core`)
- **Last meaningful commit:** v1.3.1 released 17 April 2026; 63 releases total
- **GitHub stars:** 317 (org 1 public repo, 38 followers)
- **Production users:** MoonPay Agents (primary), OpenClaw integrations
- **Commercial backing:** MoonPay Inc. (Series C, ~$3.4B valuation); 20 named contributing orgs [2]
- **Release signing:** GitHub-verified GPG (key B5690EEEBB952194)
- **Ecosystem traction:** adapters for viem, Solana, Tether WDK; cites CAIP, BIP, ERC-4337, W3C Universal Wallet, Solana Wallet Standard as prior art

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | Keys never leave `~/.ows/`; MoonPay CLI explicitly "keys never leave your machine" [10]; Ledger further isolates [2]. |
| N2 | Autonomous (no project-operated backend) | partial | **OWS spec**: yes, zero network dependency for signing [1]. **MoonPay Agents product**: requires `mp login` against MoonPay servers and uses MoonPay for on-ramp/market data [8]. |
| N3 | No central server for keys/policy/history | partial | OWS: yes. MoonPay Agents: account auth + telemetry exists; CLI is commercial. |
| N4 | Open source, permissive licence | yes | MIT [1, 2]. |
| N5 | JSON-default output | partial | SDKs return typed objects; CLI JSON output shape is not normatively fixed by the spec (conformance profile covers artifacts, not CLI output). |
| N6 | Testnet/mainnet parity | yes | Chains addressed via CAIP-2 per network (`eip155:8453` vs `eip155:84532`); no ambient network default. |

OWS-the-spec clears the non-negotiables as a design reference. MoonPay Agents-the-product does not (N2/N3) and is useful only as a commercial exemplar.

## 9. What to adopt

1. **Seven-sub-spec modularisation** (storage, signing, policy, access, key isolation, lifecycle, chains). Each adoptable independently; vendors MUST declare which profiles they implement and cannot claim blanket "compliance" [11]. Serves A5, A7 — lets us ship Storage+Signing+SorobanChainProfile first, policy later, without false advertising.
2. **Token-as-capability design** (`ows_key_...`): API token both authenticates *and* is the HKDF input that re-derives the vault-secret decryption key [3]. Revoking the token revokes decryption capability, not merely a login session. Directly counters **T1** (exfil of token alone ≠ exfil of vault) and **T4** (sub-agent token scope is cryptographically bounded, not advisory).
3. **AND-combined declarative rules + fail-closed executable pipeline** with 5-second subprocess timeout, fail-closed on missing-binary/timeout/non-zero/malformed-output [3]. Baseline pattern for our policy engine; directly addresses **T2**.
4. **Append-only JSONL audit log** of every policy decision including the draft it denied [3, 7]. Supports A1's auditability requirement and **T2/T4** forensic needs.
5. **Vault-secret re-encryption per API key** so each delegated token has its own KDF-derived wrap. This is the cryptographic form of **T4**-resistant delegation that the threat model demands.
6. **Explicit profile-declaration discipline** (spec §00): no silent degradation, optional features MUST fail clearly [11]. Matches our **T3/T10** "structurally impossible" requirement.
7. **Interactive-only mnemonic entry** (stdin, never CLI arg) during import/recovery to avoid shell-history exposure [7]. Hygiene pattern for A7.
8. **Secure-delete on `wallet delete`** (random-byte overwrite before unlink) [7]. Low-cost, addresses **T1** passively.
9. **Conformance test vectors** as a spec artifact [11]. We should publish matching vectors for Stellar-specific behaviour so downstream SDKs can self-verify.

## 10. What to avoid

1. **Policy engine lacks native amount caps, spend limits, destination allowlists, and rate limits** — all pushed to user-installed executables [3]. This fails **T2** (prompt-injected transaction) and **T6** (runaway loop) out of the box: a default deployment has no amount bound. A1's "Must-have: enforceable spending limits" is unmet by the declarative surface. Our spec must make per-tx, per-period, and per-counterparty caps first-class built-ins, not plugin territory.
2. **No on-chain policy primitive.** Delegation is entirely client-side token scope [3]. A compromised host with an unrevoked token is full token-scope loss — fails **T1** catastrophically and **T4** when revocation lags a compromise. For A2 (multi-agent orchestrator) this is the wrong model; we need Soroban smart-account policy or auth-entry-scoped delegation as the delegation primitive, not API tokens.
3. **Revocation is instant-only at next call; no chain-side revocation.** A signed-but-unbroadcast transaction from a revoked token remains valid. A2's uncomfortable failure mode ("revocation lags a compromise") is not mitigated.
4. **Ledger integration is product-only (MoonPay Agents), not normative OWS** [2]. A spec candidate claiming "hardware-backed" should bake it into the signing profile. We should make hardware-sign a normative signing profile, not vendor-specific.
5. **No approval-channel specification.** Nothing in OWS addresses **T9** (approval-channel manipulation). Signing is a function call; there is no notion of a human-confirmation channel distinct from the agent-facing surface. For A4 this is a gap.
6. **Passphrase via environment variable is a documented delivery method** [6]. The spec itself flags it as weakest but normalises it. Our spec should forbid it for unattended flows with non-trivial scope.
7. **Subprocess-enclave key isolation is future work, not current** [6]. The in-process signing path means any code in the parent Node/Python process can touch decrypted key memory for up to 30s. **T8** (supply-chain) is only weakly mitigated.
8. **Nonce management deferred to caller** [4]. Fine for EVM but a footgun for Stellar sequence numbers; A5 (CI/CD) needs idempotent sequence handling baked in, not delegated.
9. **Stellar is not supported** [1, 2]. Adding it is not a core change per doc 07, but the signing interface does not accommodate Soroban auth entries: `sign()` takes "hex-encoded transaction bytes" [4], which does not fit the `SorobanAuthorizationEntry` / `SorobanCredentials` flow where pre-auth happens out-of-band from the envelope. `signTypedData` is EVM-specific. There is no analogue of SEP-10 challenge-response, SEP-24 interactive flows, SEP-6/45 deposit-withdraw state, or SEP-41 asset metadata anywhere in the spec. Classic Stellar multisig (per-operation signer weights, thresholds) has no expression — OWS assumes single-signer-per-account. Smart accounts (Soroban `__check_auth`) would require either a new signing-interface method or an extension profile. **Adopting OWS for Stellar means: new chain profile (trivial), new signing method for auth entries (non-trivial), new policy rule types for SEP-10 identity and memo enforcement (addresses T5), and either forking the spec or submitting a Stellar signing extension upstream.**

## 11. Open questions

- Does MoonPay Agents' Ledger integration support arbitrary-contract signing or only hard-coded EVM payloads? (Not in public docs.)
- What is the upstream governance model — BDFL, committee, RFC? The standard site lists orgs but no governance doc was found.
- Do any of the 20 listed contributing orgs have in-flight Stellar/CAIP extension proposals? No public issue/PR on this found.
- Is a normative MCP tool-set profile planned, or will MCP remain product-vendor territory?
- What is the scrypt work factor ceiling before unlock becomes UX-hostile on low-power devices (A4 phones)?

## 12. Sources

1. `https://github.com/open-wallet-standard/core` — README, repo structure, supported-chains table, CLI surface, SDK snippets (accessed 2026-04-18).
2. `https://www.prnewswire.com/news-releases/moonpay-open-sources-the-wallet-layer-for-the-agent-economy-302722116.html` — launch press release, 23 March 2026; full contributor list; Ledger claim; supported chain list (2026-04-18).
3. `https://raw.githubusercontent.com/open-wallet-standard/core/main/docs/03-policy-engine.md` — rule types, evaluation pipeline, subprocess timeout, fail-closed semantics (2026-04-18).
4. `https://raw.githubusercontent.com/open-wallet-standard/core/main/docs/02-signing-interface.md` — `sign`, `signAndSend`, `signMessage`, `signTypedData`, `signHash`, `signAuthorization`; nonce-management gap (2026-04-18).
5. `https://raw.githubusercontent.com/open-wallet-standard/core/main/docs/04-agent-access-layer.md` — three access profiles, six must-preserve capabilities (2026-04-18).
6. `https://raw.githubusercontent.com/open-wallet-standard/core/main/docs/05-key-isolation.md` — mlock TTL 30s, zeroisation, passphrase-delivery tiers, subprocess-enclave future work (2026-04-18).
7. `https://raw.githubusercontent.com/open-wallet-standard/core/main/docs/01-storage-format.md` and `06-wallet-lifecycle.md` — AES-256-GCM, scrypt min N=2^16, Keystore-v3 extensions, import/export/recover/delete flows (2026-04-18).
8. `https://support.moonpay.com/en/articles/586583-moonpay-cli-for-ai-agents` — MoonPay CLI, `mp mcp` stdio server, command catalogue (2026-04-18).
9. `https://raw.githubusercontent.com/open-wallet-standard/core/main/docs/07-supported-chains.md` — chain-addition procedure, CAIP-2 enforcement (2026-04-18).
10. `https://www.moonpay.com/newsroom/open-wallet-standard` — MoonPay product-side narrative, "private key is never exposed to the agent, the LLM, or any parent process" (2026-04-18).
11. `https://raw.githubusercontent.com/open-wallet-standard/core/main/docs/00-specification.md` and `08-conformance-and-security.md` — RFC-2119 language, profile-declaration discipline, conformance test vectors, schema-version rejection (2026-04-18).

## 13. Cross-links

- Compare to `research/external/crypto/kraken-cli.md` for CLI shape and JSON-envelope discipline.
- Compare to `research/external/crypto/safe-modules.md` for on-chain policy / delegation primitives that OWS lacks.
- Compare to `research/external/crypto/turnkey.md` for the opposite custody trade-off (OWS local-first vs. Turnkey TEE-hosted).
- Feeds `06-option-a-cli-first.md` §7 and `07-option-b-agent-native.md` §7: Soroban auth-entry support requires signing-interface extension beyond OWS's `sign(tx-bytes)`; the three-surface (CLI + SDK + MCP) equal-rank pattern is directly applicable.
- Surfaces requirement candidates: (a) per-tx amount caps as first-class policy, (b) Soroban-auth-entry signing method, (c) on-chain delegation primitive via smart accounts, (d) wallet-owned approval channel for A4.
