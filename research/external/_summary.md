# External References: Cross-Cutting Summary

**Status:** tier-1 and tier-2 research waves complete — 2026-04-18.
**Depends on:** individual files under `research/external/{crypto,non-crypto,stellar-ecosystem}/` (tier 1) and `research/external/_tier-2/` (tier 2).

---

## Purpose

Cross-cutting rollup of the per-candidate analyses. Exists so a reader can compare candidates without opening every file. The summary is *derived* — facts live in the per-candidate files. If summary and source disagree, the source wins.

## Tier structure

- **Tier 1 (14 candidates)** — first research wave plus the smart-account-kit entry added in the 2026-04-20 SAK + KMP review. Drives the decision matrix in `analysis/09-decision-matrix.md` directly.
- **Tier 2 (6 candidates)** — second wave, complete. Refines tier-1 findings; no tier-1 candidate was promoted to replace a tier-2 selection.

**Total: 20 external candidates researched, ~400-800 words each, all citing primary sources.**

---

## Tier 1 — crypto candidates

| Candidate | Status | Agent interface | Custody | Policy locus | Delegation | N1 | N2 | N3 | Licence | Adoption |
|---|---|---|---|---|---|---|---|---|---|---|
| Kraken CLI | done | CLI + MCP stdio | fully custodial (Kraken) | vendor-side (API key scopes) | vendor subaccounts + shared keys | no | no | no | MIT | Kraken-built; integrations for Claude, Cursor, Codex, Copilot, Gemini, Goose |
| MoonPay Agents + OWS | done | CLI + Node/Python SDK + MCP stdio | local AES-256-GCM vault (`~/.ows/`); product adds MoonPay login | local — 3 declarative rules + user-installed executable | scoped API tokens with HKDF re-wrap (capability-bound) | yes (spec) / partial (product) | partial | partial | MIT | 20 backing orgs (PayPal, Ethereum Foundation, Circle, …); 317 ★ |
| Trust Wallet Agent Kit (TWAK) | done | CLI + MCP stdio + REST (one binary) | Mode A: local AES-256-GCM HD wallet. Mode B: WalletConnect to user's TW mobile | per-invocation CLI flags (no persistent rule set) | none | partial (A) / yes (B) | no | no | MIT (skills); CLI distributed as minified npm bundle only | 220M TW installs; 32 MCP tools |
| Coinbase Agentic / CDP Wallets | done | CLI (`awal`) + MCP + REST | CDP-operated AWS Nitro Enclave; keys never leave TEE | vendor-side enclave (typed criteria) | sub-organizations + ERC-4337 spend permissions (Base only) | no (self-custodial branding only) | no | no | proprietary service; skills MIT; AgentKit Apache | Coinbase; Questflow, FereAI |
| Turnkey | done | REST + TS/Rust/Dart SDKs | Turnkey-held inside AWS Nitro Enclaves (QuorumOS) | vendor-side enclave DSL `{effect, consensus, condition}` | sub-organizations + expiring session keys | no | no | no | proprietary service; SDKs Apache-2.0; QuorumOS AGPL-3.0 | Moonshot, Spectral Labs |
| Safe (+ modules) | done | on-chain contract calls; Safe{Wallet} optional UI | self-custodial (owner EOAs / hardware); modules add second custody tier | on-chain per-module (AllowanceModule caps, Zodiac Roles) | module attachment + module-internal policy | yes | yes | yes | LGPL-3.0 (partial on N4) | most major DAO treasuries; Gnosis Pay routes through Zodiac Roles |

## Tier 1 — Stellar-ecosystem candidates

| Candidate | Status | Agent interface | Custody | Policy locus | Delegation | N1 | N2 | N3 | Licence | Adoption |
|---|---|---|---|---|---|---|---|---|---|---|
| Meridian Pay (SDF) | done | browser UI only; admin scripts only | user passkey on Soroban smart account + **platform recovery cosigner** (SDP) | none on-chain — `__check_auth` ignores `_auth_contexts` | none | partial | no | no | Apache-2.0 (contract) / ISC (demo) | ~1,000 Meridian attendees; SDF states "not long-term supported" |
| Stellar MCP (christopherkarani) | done | MCP stdio only (5 tools) | plaintext secret file + `secretKey` as tool argument per call | none | none | partial | no | no | ISC | 0 stars (fork); upstream pivoted |
| Freighter + Wallet Kit | done | browser extension + TS package (browser-only) | self-custodial (extension local storage) | per-transaction popup only; no caps / rate limits | none at wallet layer | yes | partial | partial | Apache-2.0 / MIT | SDF reference; `lab.stellar.org`, Blend, FXDAO, xBull, Soroban Domains |
| stellar-cli | done | CLI only (no MCP; pub Rust library crate) | self-custodial (TOML + optional OS keyring + Ledger) | none | none (HD paths only) | yes | yes | yes | Apache-2.0 | SDF incumbent CLI |
| Smart Account Kit (SAK TS + KMP `OZSmartAccountKit`) | done | two client SDKs: TypeScript (web-only; Vite + React demo with Freighter extension integration) and Kotlin Multiplatform (Android / iOS / macOS / Web; 7-screen demo with Reown WalletConnect v2) driving the same OZ `stellar-accounts` v0.7.1 contract | OS-level passkey (WebAuthn) + optional delegated G-address or external ed25519 signers for agent paths | on-chain (OZ `ContextRule` + policy); client does not implement policy | OZ-native context rules + scoped policies | yes | **partial** (KMP does on-chain scan for rule enumeration; SAK and reverse lookup in both SDKs default to a hosted Cloudflare Worker on the `sdf-ecosystem.workers.dev` subdomain — liveness dependency, not data exposure) | **yes** (indexer reads only public on-chain event data; query-pattern side channel is a traffic-analysis concern, not a data-exposure concern) | both Apache-2.0 | SAK used by OZ stellar-contracts demos; KMP maintained as part of Soneso SDK line |

Additional non-negotiable flags: **stellar-cli** fails N5 (`--output json` on only ~9 of ~50 commands); **Stellar MCP** fails N6 (testnet Horizon URL hardcoded). **Smart Account Kit (SAK + KMP)** default hosted-indexer URL is a third-party liveness dependency when shipped unchanged (SAK's rule enumeration depends on it; KMP's does not). The indexer itself holds no private data (it reads public on-chain events), so N3 is not at risk on data-exposure grounds. Wallets that need to avoid the liveness dependency either self-host the indexer, use KMP's on-chain rule scan, use deterministic-address derivation for contract discovery, or accept the dependency.

## Tier 1 — non-crypto design references

Non-negotiable columns do not apply.

| Candidate | Interface shape | Output model | Config / identity | Extensibility | Lead patterns to take |
|---|---|---|---|---|---|
| `gh` (GitHub CLI) | CLI (+ `go-gh` library) | `--json <fields>` + `--jq` + `--template`; self-describing field discovery; **no JSON envelope even in `--json` mode** | `$GH_CONFIG_DIR` (XDG); `GH_TOKEN` env > OS keyring; dedicated exit code `4` for auth-required | `gh extension install` (no signature verification) | `--json <fields>` + `--jq` + template triad; self-describing field discovery; keyring-first with `--insecure-storage` opt-out; auth-required exit code |
| Stripe CLI | CLI (resource-verb taxonomy + generic HTTP escape hatch) | bare JSON on stdout; no envelope; errors on stderr; `--confirm` prompt on `delete` | TOML at `~/.config/stripe/config.toml`; live keys → macOS Keychain; `--project-name`/`-p` profiles | `stripe plugin install` (no documented sandbox) | resource-noun + verb; `--idempotency` on every mutation; `stripe listen --forward-to` subprocess pattern for event streams; `stripe trigger` fixtures; named profiles |
| AWS CLI | CLI only; no MCP | bare JSON/YAML/text/table; **no uniform envelope**; per-service drift; `--query` (JMESPath) | two-layer: `~/.aws/config` + `~/.aws/credentials`; named profiles; documented precedence chain | `credential_process` external-signer hook | two-layer config split with same profile name; documented single-source precedence chain; `credential_process`-style signer hook; `--output json` + `--query` |

---

## Tier 2 — crypto candidates

| Candidate | Status | Agent interface | Custody | Policy locus | Delegation | N1 | N2 | N3 | Licence | Adoption |
|---|---|---|---|---|---|---|---|---|---|---|
| Privy | done | SDK + shipped agent Skill + MCP | 3 modes: embedded (Privy-held), 2-share device+server reconstitution, delegated (TEE re-provisioning). All variants: signing via api.privy.io liveness. | vendor-side (api.privy.io); `raw_sign` for Stellar (no Soroban/memo/SEP awareness) | TEE re-provisioning, not key sharing | partial | no | no | proprietary service | Stripe acquisition (Oct 2025); large embedded-wallet deployments |
| Fireblocks | done | SDK + REST + web console | MPC 2-of-3: vendor holds 2 shares inside SGX enclaves | SGX-enforced TAP DSL (rules + AuthorizationInfo quorums); Admin-Quorum governance plane | role/group membership (config, not crypto) | no | no | no (N4 also no — closed) | proprietary; closed source | institutional market leader; regulated custody |
| MetaMask Snaps | done | browser extension + Snap JSON-RPC; not a deployment model | (out of scope — this is an extensibility model) | per-Snap permission manifest; SES sandbox; install-time consent only (no first-invoke gate) | none (Snaps are extensions, not signers) | n/a | n/a | n/a | proprietary extension; Snap SDK MIT | widespread; Snap registry with ~audit-tier entries |
| Tether WDK / tether.wallet | done | SDK + MCP (annotated tools + elicitation) + file-based AgentSkills | local BIP-39 seed (env var `WDK_SEED` or constructor arg); no WDK vault format; embedding app provides storage | **none at SDK or MCP layer**; READ/WRITE tool-list separation is the only scoping vector | none (same seed = N BIP-44 accounts; no authority scoping) | yes | partial (Tether-operated indexer + `name@tether.me` resolver on happy path) | partial | Apache-2.0 | Tether-backed; consumer `tether.wallet` (app closed-source); WDK open-sourced 2025-10 |
| Phantom MCP server | done | MCP stdio only (26 tools); thin wrapper over `@phantom/cli` `--mcp` | hosted KMS at api.phantom.app; session credential (Ed25519/P-256 stamper) on client | backend-only (not user-configurable via MCP); advisory `confirmed: false → true` preview flow | HD `derivationIndex` only; coarse revocation via account portal | no | no | no | proprietary service; CLI npm MIT | Phantom's first-party product |

## Tier 2 — Stellar-ecosystem candidates

| Candidate | Status | Agent interface | Custody | Policy locus | Delegation | N1 | N2 | N3 | Licence | Adoption |
|---|---|---|---|---|---|---|---|---|---|---|
| LaunchTube | done | HTTP service with JWT bearer | server-held `FUND_SK`; sequence-pool keys deterministic from master; operator sees plaintext envelopes | none; credit-as-capability accounting with eager→bid→final spend | none; JWT carries no scope | yes (for user's signed envelopes) | no (hosted) / partial (self-hosted) | no | **no** (no LICENSE; `package.json private:true`) | SDF-operated; **officially being discontinued** in favour of OpenZeppelin Relayer (Channels Plugin) |

**Tier-2 additional non-negotiable observations:**

- **No tier-2 crypto candidate passes all of N1 / N2 / N3.** Tether WDK is the closest (N1 yes, N2/N3 partial) but the partiality comes from mandatory Tether infrastructure on the happy path.
- **LaunchTube's disqualification is architectural, not situational.** Self-hosting does not remove the operator-sees-envelopes property; only removing the third-party submitter entirely does. SDF's announced deprecation confirms this model is ending.

---

## Disqualifying failures of non-negotiables

All crypto candidates (tier 1 + tier 2) whose deployment model fails N1, N2, or N3. Useful for the RFP narrative about why the market does not solve this on Stellar.

| Candidate | Wave | Failure | Explanation |
|---|---|---|---|
| Kraken CLI | 1 | N1 / N2 / N3 no | Exchange CLI; funds and keys on Kraken; CLI holds only API credentials. |
| Coinbase Agentic / CDP | 1 | N2 / N3 no | Every signature requires `api.cdp.coinbase.com` and a Coinbase-operated AWS Nitro Enclave; encrypted keys and policies live in CDP's database and portal. "Self-custodial" is branding. |
| Turnkey | 1 | N1 / N2 / N3 no | Keys generated and held server-side inside Turnkey's Nitro Enclaves; user holds only an authenticator, not key material. Turnkey outage or operator coercion blocks signing. |
| MoonPay Agents (product) | 1 | N2 / N3 partial | `mp login` requires MoonPay servers; CLI uses MoonPay for on-ramp and market data. **OWS spec clears** — the product is the disqualified artefact. |
| Trust Wallet Agent Kit | 1 | N2 / N3 no; N6 no | All REST calls route through `tws.trustwallet.com`; no testnet parity documented for non-EVM chains. Mode B (WalletConnect to user's mobile Trust Wallet) clears N1 but still depends on TW infrastructure. |
| Meridian Pay | 1 | N1 partial; N2 / N3 no | SDF-operated SDP backend, wallet-backend fee-sponsor, and platform-held recovery cosigner are all mandatory paths. SDF scopes it as "not long-term supported." |
| Stellar MCP server | 1 | N1 partial; N2 / N3 / N6 no | Plaintext `agent-keys.txt` committed; `secretKey` passed as tool argument every call; hard dependency on Launchtube + Mercury; testnet Horizon URL hardcoded. |
| Freighter | 1 | N2 / N3 partial | Keys self-custodial, but production extension is configured against `freighter-backend-prd.stellar.org` for indexer/balance/memo-required data by default. |
| Privy | 2 | N1 partial; N2 / N3 no | Three custody modes; all three require liveness to `api.privy.io` for signing. 2-share mode "collapses" on delegation (delegated-signer mode moves share custody back to Privy). |
| Fireblocks | 2 | N1 / N2 / N3 / N4 no | MPC 2-of-3 with vendor holding 2 shares inside SGX enclaves; TAP policy enforced vendor-side only; closed source (no self-hosted deployment). |
| Tether WDK | 2 | N2 / N3 partial | Local seed is self-custodial, but Tether-operated indexer and `name@tether.me` resolver are on the happy path; WDK has zero policy primitives. |
| Phantom MCP server | 2 | N1 / N2 / N3 no | Hosted KMS at `api.phantom.app`; client holds only an OAuth-derived session stamper; equivalent to Coinbase CDP / Turnkey's custody category. |
| LaunchTube | 2 | N2 / N3 / N4 no | Third-party submitter sees plaintext envelopes correlated to JWT `sub`; server-held `FUND_SK`; no LICENSE / `package.json private:true`. **Being discontinued by SDF.** |

**Candidates that pass N1 / N2 / N3 cleanly:** **Safe** (tier 1 — LGPL-3.0 so N4 partial; EVM-only) and **stellar-cli** (tier 1 — N5 partial, JSON on only ~9 of ~50 commands).

Observation: **among 11 tier-1 + tier-2 crypto candidates with a deployment story, only two satisfy all three custody-related non-negotiables**. Safe is not on Stellar; stellar-cli lacks the agent affordances our actors need. This is the market gap our proposal fills.

---

## Consolidated "what to adopt"

Lifted from §9 of each candidate file; each entry cross-links to source. **Grouped by axis, with tier-2 contributions marked `[T2]`.**

### Custody and key handling

- **Token-as-capability via HKDF re-wrap of the vault secret** (`moonpay-agents-ows.md` §9) — revoking a token revokes decryption authority, not just auth. **T1**, **T4**.
- **OS-keyring-first credential storage with explicit insecure-storage opt-out** (`gh-cli.md` §9 / `stellar-cli.md` §9) — default Keychain/DPAPI/Secret Service; degrade only under a named flag. **T1**, serves **A1**, **A5**.
- **`credential_process`-style external-signer hook** (`aws-cli.md` §9) — a config line naming an executable that returns credentials-with-expiry on stdout. Exactly the shape for a hardware-wallet / passkey / TEE signer plugin. **A5**.
- **Hardware-wallet abstraction via `Signer`/`SignerKind` enum** (`stellar-cli.md` §9) — Rust extension point for custom backends (passkey smart accounts). **T1**.
- **Secret-handling discipline** (`kraken-cli.md` §9) — `--api-secret-stdin` / `--api-secret-file`, 0600 config perms, warning on command-line secret use. **T1**.
- **`[T2]` 2-share device+server reconstitution as optional hosted mode** (`_tier-2/crypto/privy.md` §9) — serves **A4** (human approval through a companion), addresses **T1** (neither share alone signs). Opt-in, not default.
- **`[T2]` Tether WDK-style `.dispose()` / `server.close()` lifecycle** (`_tier-2/crypto/tether-wdk.md` §9) — explicit clear-seed-from-memory primitives. Addresses **T1**.

### Policy and approval

- **Typed-criteria policy schema** (`coinbase-agentic.md` §9) — `ethValue` / `evmAddress` / `evmData` / `netUSDChange` / `evmNetwork` with first-match stop-semantics. **T2**, **T3**, **T4**.
- **USD-denominated criterion (`changeCents`)** (`coinbase-agentic.md` §9) — avoids asset-unit confusion. **T3**.
- **Compact `{effect, consensus, condition}` triple** (`turnkey.md` §9) — over `(activity.type, activity.resource, activity.action)`.
- **Policy engine as a locally-evaluated pre-sign gate** (`moonpay-agents-ows.md` §9 / `safe.md` §9) — in the signer process or on-chain. Vendor-side is not acceptable.
- **Module-attachment as the delegation primitive** (`safe.md` §9) — extend OZ `Policy` trait so policies can *initiate* execution, not only *enforce* one. **A2**, **A4**, **T4**.
- **On-chain module-attestation registry** (`safe.md` §9) — mirror ERC-7484/7512 (Rhinestone Registry): signed attestations keyed on WASM hash. **T8**.
- **Guard trait with pre/post hooks** (`safe.md` §9) — `checkTransaction` / `checkAfterExecution` / `checkModuleTransaction`. **T2**, **T4**.
- **Fail-closed executable policy plugin** with 5-second timeout and JSONL audit log (`moonpay-agents-ows.md` §9) — for rules the declarative layer cannot express.
- **Dangerous-tool gate requiring `acknowledged=true`** with fail-closed argv filter (`kraken-cli.md` §9). **T2**, **T8**.
- **Per-invocation safety flags** (`trust-wallet-agent-kit.md` §9) — `--max-usd`, `--confirm-to <addr>`, `--confirm-unlimited`.
- **`[T2]` `AuthorizationInfo` approval object** (`_tier-2/crypto/fireblocks.md` §9) — per-group thresholds with AND/OR combiner and `allowInitiatorAsApprover` flag. Richer than Turnkey's `{effect, consensus, condition}` triple. **T4**, **T9**; **A2**, **A4**.
- **`[T2]` Typed source/destination object kinds** (`_tier-2/crypto/fireblocks.md` §9) — `INTERNAL_WALLET` / `EXTERNAL_WALLET` / `CONTRACT` / `ONE_TIME_ADDRESS` / `NETWORK_CONNECTION`. Extends Coinbase CDP's raw-address approach; maps to Stellar's G-account / C-account / SEP-10-identity distinctions. **T5**, **A3**.
- **`[T2]` Admin-Quorum governance plane** (`_tier-2/crypto/fireblocks.md` §9) — separate quorum gating policy mutations, allowlist changes, and user additions. Policy edits themselves require n-of-m signatures. **T2**, **T8**; **A2**, **A7**.
- **`[T2]` Declarative `initialPermissions`-style manifest** (`_tier-2/crypto/metamask-snaps.md` §9) — narrow typed-capability taxonomy; caveat-able nouns only, no free-form grants. **T8**; **A1**, **A2**.
- **`[T2]` SES-Compartment-per-skill sandbox** (`_tier-2/crypto/metamask-snaps.md` §9) — per-skill private storage + mandatory `maxRequestTime` + **first-invoke gate on signing capabilities** (closes the install-only consent gap). **T2**, **T8**.
- **`[T2]` Mandatory third-party audit + signed WASM / source-hash attestation registry** for any skill requesting signing or key-derivation (`_tier-2/crypto/metamask-snaps.md` §9) — mirrors Safe's Rhinestone Registry finding; `(package, version, shasum)`-pinned installs. **T8**, **A7**.
- **`[T2]` `owner_id` with mutation-signed-by-owner** on every policy resource (`_tier-2/crypto/privy.md` §9) — policy edits are themselves cryptographically bound to the owner, not to session state. **T1**, **T8**, **A7**.
- **`[T2]` Delegation-as-TEE-re-provisioning** (`_tier-2/crypto/privy.md` §9) — when user grants authority to a sub-agent, re-provision key material in the sub-agent's TEE/vault rather than sharing key material. **A2**, **A4**, **T4**.

### Delegation and approval channel

- **WalletConnect as the approval-channel transport** (`trust-wallet-agent-kit.md` §9) — structurally defends **T9**; serves **A4**. Must ship methods (`signAuthEntry`, `signMessage`) that Creit's current WC module rejects.
- **Custom `__check_auth` WebAuthn baseline** (`meridian-pay.md` §9) — ~100 LOC starting point for our passkey signer, to extend with policy. **T2**, **T4**.
- **Router / multicall contract** (`meridian-pay.md` §9) — bundle invocations under one approval. Reduces habituation against **T9**; serves **A4**.
- **Deterministic-address-from-salt + OZ `stellar-merkle-distributor`** (`meridian-pay.md` §9) — the only Meridian Pay primitive that generalises to **A2** agent delegation (bounded *inbound* budgets).
- **SEP-43 verbatim + Wallet Kit module** (`freighter-walletkit.md` §9) — drops into every Stellar dapp already using Wallets Kit.
- **Root-quorum bypass with explicit unbrick semantics** (`turnkey.md` §9) — break-glass primitive for **A7**.
- **Expiring session keys** (`turnkey.md` §9) — default 15 min, max 10 concurrent, non-exportable. **T1**, **T4**.

### Agent-facing CLI and MCP ergonomics

- **`--json <fields>` + `--jq <expr>` + `--template <tmpl>` triad** (`gh-cli.md` §9) with self-describing field discovery. **A1**-**A2**; realises **N5**.
- **Resource-noun + verb taxonomy + generic escape hatch** (`stripe-cli.md` §9) — `stellar accounts create` + `stellar tx submit`.
- **First-class `--idempotency` flag on every mutation** (`stripe-cli.md` §9) — deterministic client-side request IDs around sequence-number collisions.
- **`stripe listen --forward-to` subprocess pattern** (`stripe-cli.md` §9) — direct analogue for Horizon SSE / Soroban RPC `getEvents`.
- **`stripe trigger` for fixture events** (`stripe-cli.md` §9) — deterministic testnet scenario generators for agent integration tests.
- **Built-in MCP server reusing the CLI dispatch path** (`kraken-cli.md` §9) — schemas generated from clap/equivalent metadata.
- **Nine-category error envelope with enriched `rate_limit` fields** (`kraken-cli.md` §9) — agents route on category, not message.
- **Uniform `{ok, data|error, request_id}` envelope** (inverse of `aws-cli.md` §10 lesson).
- **Standardised exit codes including dedicated auth-required code (`4`)** (`gh-cli.md` §9).
- **Structured stderr audit log** with agent/instance/pid/transport fields and argument *keys only* (`kraken-cli.md` §9).
- **Structurally distinct test/live mode tagged in every response** (`kraken-cli.md` §9 paper mode / `stellar-cli.md` §9 `--network` required). **T10**.
- **`[T2]` MCP tool annotations** — `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` on every tool + elicitation-based per-call confirmation (`_tier-2/crypto/tether-wdk.md` §9; also `_tier-2/crypto/phantom-mcp-server.md` §9 confirms). **A1-A4**, partial **T9**.
- **`[T2]` `WALLET_READ_TOOLS` vs `WALLET_WRITE_TOOLS` typed export groupings** (`_tier-2/crypto/tether-wdk.md` §9) — scope pre-selection at tool-registration time. **A6**, read-mostly actors never see write tools.
- **`[T2]` Typed amount parsing with typed error codes** (`_tier-2/crypto/tether-wdk.md` §9) — `parseAmountToBaseUnits` emits `EMPTY_STRING` / `INVALID_FORMAT` / `EXCESSIVE_PRECISION`. Direct **T3** defence.
- **`[T2]` Dual-unit amount schema** (`_tier-2/crypto/phantom-mcp-server.md` §9) — `amountUnit: "ui" | "base"` + explicit `decimals` on every amount field. **T3**.
- **`[T2]` Two-step `confirmed: false → true` flow** (`_tier-2/crypto/phantom-mcp-server.md` §9) — *but* `confirmed` must be a **wallet-issued nonce** returned by the simulation step, not an agent-passed free boolean. The pattern as shipped by Phantom is advisory; nonce-bound makes it an actual T2/T3 defence.
- **`[T2]` Chain-namespaced tool names (`stellar_*`) with CAIP-2 on every schema** (`_tier-2/crypto/phantom-mcp-server.md` §9) — structural defence against **T10** and cross-chain confusion.
- **`[T2]` `simulate` as first-class peer of `send`** (`_tier-2/crypto/phantom-mcp-server.md` §9) — not a flag; a tool. **A6**.

### Configuration and identity

- **Two-layer config** (`aws-cli.md` §9) — non-secret TOML + OS-keyring secrets keyed by the same profile name.
- **Documented single-source credential precedence chain** (`aws-cli.md` §9).
- **`AWS_PROFILE`-style env var as selector** (`aws-cli.md` §9) — mirror as `STELLAR_WALLET_PROFILE`.
- **Identity TOML per identity under `$XDG_CONFIG_HOME/stellar/identity/`, `0o700`** (`stellar-cli.md` §9).
- **Named profiles over TOML** (`stripe-cli.md` §9).

### Extension, observability, integration

- **Stellar-cli external-binary plugin pattern** (`stellar-cli.md` §9) — any `stellar-<name>` on `PATH` becomes `stellar <name>`. **Center-of-gravity lever for option A.**
- **`soroban_cli` public Rust library crate** (`stellar-cli.md` §9) — embed key derivation, identity files, Ledger signer, Soroban tx assembly, SEP-5 HD paths.
- **Seven-sub-spec modular profiles** (`moonpay-agents-ows.md` §9) — wallets MUST declare which profiles they implement; no blanket "compliance."
- **MCP resource pattern for agent-facing documentation** (`stellar-mcp-server.md` §9).
- **Signer-shape routing** (`stellar-mcp-server.md` §9) — inspect `needsNonInvokerSigningBy()` to choose classic keypair vs. smart-wallet signing.
- **Simulation-before-submit branch** (`stellar-mcp-server.md` §9).
- **`[T2]` Sequence-pool with deterministic `SHA-256(master || index)` derivation** (`_tier-2/stellar-ecosystem/launchtube.md` §9) — the core LaunchTube primitive lifted into our **in-process** pool. **A1**, **A2**.
- **`[T2]` Credit-as-capability accounting with eager → bid → final spend** (`_tier-2/stellar-ecosystem/launchtube.md` §9) — bounded-throughput pattern for fee sponsorship. **A1**, **A3**.
- **`[T2]` Simulation-auth-matches-submission-auth audit** (`_tier-2/stellar-ecosystem/launchtube.md` §9) — single-op typing with auth-entry equivalence check. **T3**.

---

## Consolidated "what to avoid"

Lifted from §10 of each candidate file.

### Custody and deployment model

- **Custodial or enclave-held-by-vendor key operations** (Kraken, Coinbase CDP, Turnkey, Phantom, Stellar MCP, Privy, Fireblocks) — incompatible with **N1 / N2 / N3**.
- **"Self-custodial" as branding for enclave-resident keys with no user-visible attestation** (Coinbase CDP, Turnkey, Phantom) — misleading against N1.
- **`[T2]` "Self-custodial" over 2-of-2 that collapses on delegation** (`_tier-2/crypto/privy.md` §10) — Privy's 2-share mode is genuinely self-custodial for direct use, but the delegated-signer mode moves the user's share to Privy. Any delegation mode must preserve the self-custody property.
- **Bearer credentials as the only auth primitive** (gh, Stripe, AWS, Kraken) — host compromise = total loss.
- **Mandatory central submitter / fee sponsor / recovery cosigner** (Meridian Pay SDP + wallet-backend + platform-held recovery; LaunchTube) — any one of these fails N2/N3.
- **Plaintext key material on disk as the default** (stellar-cli `keys generate`; Stellar MCP `agent-keys.txt`; **`[T2]` Tether WDK `WDK_SEED` env-var / constructor-arg happy path**).
- **Hardcoded testnet URLs / ambient network default** (Stellar MCP; Stripe implicit test/live). **T10**.
- **No hardware-signing path** (Kraken, Stripe, AWS, default stellar-cli).
- **`secretKey: z.string()` tool arguments** (Stellar MCP) — raw `S...` secrets crossing the agent trust boundary every call.
- **`[T2]` MPC split where the vendor holds 2-of-3** (`_tier-2/crypto/fireblocks.md` §10) — N1/N2 failure; user cannot sign or recover independently.
- **`[T2]` Hosted KMS with OAuth session as client credential** (`_tier-2/crypto/phantom-mcp-server.md` §10) — session file exfil = full hosted-wallet scope, despite not being a raw private key.

### Policy and enforcement

- **No local policy layer at all** (gh, Stripe, AWS; Kraken, stellar-cli, Freighter, Stellar MCP on Stellar; **`[T2]` Tether WDK at SDK and MCP layers**; **`[T2]` Phantom MCP**).
- **Vendor-dashboard policy** (Coinbase CDP, Stripe restricted-key scopes, Turnkey policy engine, Fireblocks TAP).
- **Policy engine with no native amount caps, allowlists, or rate limits** (MoonPay OWS declarative layer) — all pushed to user-installed executables. Ship executables as an extension, not the first answer.
- **Single passkey + `_auth_contexts` ignored** (Meridian Pay).
- **`--allow-dangerous` as a coarse all-or-nothing autonomy switch** (Kraken MCP).
- **`[T2]` Install-time consent as the only extension-authorization gate** (`_tier-2/crypto/metamask-snaps.md` §10) — fails **T2** for network + signing capabilities.
- **`[T2]` `endowment:long-running`-style unbounded background execution** (`_tier-2/crypto/metamask-snaps.md` §10) — deprecated only after audit findings; ship caps as non-optional. **T6**.
- **`[T2]` Pre-canonicalisation argument validation** (`_tier-2/crypto/metamask-snaps.md` §10) — OtterSec `toJSON()` property-spoofing bypass. Canonicalise first, freeze, then assert. **T2**, **T8**.
- **`[T2]` Advisory-only `confirmed` boolean the agent can set to `true` on first call** (`_tier-2/crypto/phantom-mcp-server.md` §10) — bypasses simulation entirely. The pattern needs a wallet-issued nonce.
- **`[T2]` `raw_sign`-only Stellar support** (`_tier-2/crypto/privy.md` §10) — no Soroban / memo / destination / SEP awareness means policy cannot reason about Stellar-specific semantics. **T2**, **T4**, **T5**.
- **`[T2]` Vendor-side SGX policy enforcement** (`_tier-2/crypto/fireblocks.md` §10) — policy does not survive vendor compromise.

### Delegation

- **No delegation primitive** — across every tier-1 crypto candidate except Safe modules and OWS tokens, and across every tier-2 candidate.
- **Delegation via shared API keys / vendor-side subaccounts** (Kraken, Stripe, Turnkey).
- **Human-gated-only approval without caps / rate limits** (Freighter).
- **`[T2]` Role/group membership as the only delegation primitive** (`_tier-2/crypto/fireblocks.md` §10) — config, not crypto; revocation is a vendor action.
- **`[T2]` Same-seed BIP-44 as "delegation"** (`_tier-2/crypto/tether-wdk.md` §10; `_tier-2/crypto/phantom-mcp-server.md` §10) — HD-derived accounts share the seed; compromise of any account = compromise of all.
- **`[T2]` Bearer-JWT capability with no scope** (`_tier-2/stellar-ecosystem/launchtube.md` §10) — token exfil = full credit-balance drain.

### Output and error handling

- **Single exit code `1` for every failure class** (Kraken).
- **Errors on stdout, not stderr** (Kraken).
- **No uniform output envelope, per-service drift** (AWS, gh in `--json` mode).
- **`--output text` that drifts per service** (AWS).
- **Human-readable default output** (Kraken, TWAK).
- **`--json` not universal** (gh; stellar-cli).
- **`[T2]` Solana-specific schema quirks as first-class tool shapes** (`_tier-2/crypto/phantom-mcp-server.md` §10) — base58/base64 tx-vs-signature split, blockhash/ALT/compute-units simulation shape. Our MCP surface must model Soroban simulation (`SorobanAuthorizationEntry` + footprint + resource fees), not copy Solana's.

### Extensibility and supply chain

- **Extension/plugin install without signature verification or sandboxing** (gh, Stripe).
- **Trusted-but-unverified modules** (Safe's own warning).
- **LGPL-3.0 for a reference implementation** (Safe).
- **Delegate-call in module execution** (Safe `operation = DelegateCall`).
- **Unremovable-guard failure mode** (Safe guards can brick an account if buggy).

### Approval channel

- **Agent-presented approval UX** — if the agent renders the prompt, **T9** is unaddressed.
- **No approval-channel spec at all** (OWS) — leaves **T9** to the product.
- **`--skip-verify` for TLS on event-forwarding** (`stripe listen`).

### Infrastructure and operations

- **`[T2]` Third-party submitter that sees plaintext envelopes correlated to a bearer token** (`_tier-2/stellar-ecosystem/launchtube.md` §10) — this is exactly what makes Meridian Pay and Stellar MCP fail N2/N3 on Stellar. **Avoid as a dependency; ship an in-process sequence pool instead.**
- **`[T2]` Server-held master funding signer with no scoping** (`_tier-2/stellar-ecosystem/launchtube.md` §10) — `FUND_SK` style single-key-drain risk.

---

## Cross-cutting strategic findings

### Tier-1 findings (stand)

1. **Only Safe and stellar-cli pass N1 / N2 / N3 cleanly** among all 11 tier-1 + tier-2 crypto candidates with a deployment story. Safe is EVM-only and LGPL; stellar-cli is Stellar-native, Apache-2.0, with a public `soroban_cli` library crate and external-binary plugin mechanism. **Option A's implementation converges concretely onto "extend stellar-cli."**

2. **The policy-engine vocabulary converges** across strong agent-oriented products: typed criteria (Coinbase CDP), `{effect, consensus, condition}` triple (Turnkey), capability-bound tokens with HKDF re-wrap (MoonPay OWS), module-attachment (Safe), `[T2]` `AuthorizationInfo` quorums + typed object kinds (Fireblocks). None cover Stellar's native primitives (Soroban auth entries, SEP-10 home-domain, memo-as-identity, multisig weights). Our policy engine must express the **union** of these vocabularies over Stellar primitives.

3. **Meridian Pay is confirmed separate from the SDF x402-MCP effort** — zero `x402` references across both SDF repos.

### New findings from tier-2

4. **LaunchTube is being discontinued by SDF** in favour of OpenZeppelin Relayer with the Channels Plugin (confirmed via `developers.stellar.org` deprecation page). Immediate implications:
   - `stellar-capabilities/08-infra-ops.md` must document this pivot.
   - Any Stellar-side candidate depending on LaunchTube (Meridian Pay, Stellar MCP) has a migration path forced on it.
   - OZ Relayer's Channels Plugin licence is likely AGPL-3.0 — if confirmed, clashes with N4 as a deployment dependency.
   - This **sharpens option A's concrete design**: ship the in-process sequence-pool as a first-class wallet feature, not as an optional LaunchTube fallback.

5. **The emerging "agent-MCP" pattern is uniform across all 2026 candidates**: CLI + MCP-stdio + REST from one binary, `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` on every tool, chain-namespaced tool names, two-step simulate-then-commit flow. **None of them enforce the commit-nonce binding** — Phantom's `confirmed` is a free boolean, Tether WDK's elicitation is disableable, TWAK's flags are agent-set. Our wallet should ship the same tool-shape discipline *with* wallet-issued nonces from day one. This is a cheap, large, and immediately visible differentiator.

6. **The attestation-registry pattern for extensions is converging independently**: Safe's ERC-7484 / Rhinestone Registry and MetaMask Snaps' `(package, version, shasum)`-pinned installs with audit-gated capabilities both reach the same architecture from different directions. Our skills system should adopt this pattern without reinvention — it is the cheapest way to defend T8 in the extensibility story.

7. **"Self-custodial" is no longer a meaningful marketing label** in the agent-wallet space. Seven of 11 tier-1+tier-2 crypto candidates claim some form of self-custody; only two (Safe, stellar-cli) pass N1/N2/N3 under our strict definitions. Our RFP narrative should articulate a **precise self-custody definition** (e.g. "the signing key is exclusively derivable by the user or their designated local-host process, with no server liveness required for any signing operation") to stand apart.

### Non-wallet findings that affect the decision

- **SEP-43 is the canonical Stellar-dapp wallet interface shape** (`freighter-walletkit.md` §6). Our wallet must implement it to be a Wallet Kit module.
- **WalletConnect v2 on `stellar:pubnet` / `stellar:testnet`** is the cross-wallet approval transport. We should ship WC v2 with `signAuthEntry` and `signMessage` (which Creit's current WC module rejects), making us more capable than the incumbent path.
- **x402 on Stellar is handled via Soroban auth entries** (confirmed in Stellar docs). The x402 flow is a narrow case of the Soroban signing surface, not a separate protocol. **MPP** (Machine Payments Protocol — Tempo + Stripe, IETF individual submission `draft-ryan-httpauth-payment`, Stellar support via `@stellar/mpp`, covered in `research/stellar-capabilities/10-mpp.md`) sits alongside x402 as a second HTTP-402-based agentic-payment protocol: MPP **charge mode** likewise maps to the Soroban auth-entry surface (client signs a `SorobanCredentialsAddress` entry for SAC `transfer`), but MPP **channel mode** introduces a distinct signing primitive — off-chain cumulative commitments signed with a dedicated ed25519 commitment key against a one-way payment-channel Soroban contract — that is NOT an auth-entry signature and requires a separate wallet-side code path.
- **LaunchTube → OZ Relayer migration** must be accounted for in `stellar-capabilities/08-infra-ops.md`.

---

## Open follow-ups from both research waves

Candidate entries' §11 lists. Drive tier-2 follow-up research and inform `analysis/03-requirements.md` candidates queue.

- **SDF x402-MCP specification** — what the agent-facing surface will look like; whether it reuses Meridian Pay's account contract or ships a policy-bearing replacement.
- **OZ Relayer Channels Plugin** — licence confirmation (AGPL-3.0?); self-hosting posture; trust-model comparison to LaunchTube.
- **Turnkey raw-payload metadata** — whether Stellar network hashes can be distinguished in policy; whether QuorumOS + core-enclaves can run end-to-end independent of Turnkey's provisioning service.
- **OWS** — governance model; any in-flight Stellar / CAIP extension proposals among the 20 contributors; whether a normative MCP tool-set profile is planned.
- **Trust Wallet Agent Kit** — dormant SLIP-44 148 (Stellar) listing in `MAINTAINED_COIN_IDS`; Stellar support planned?
- **Safe** — per-enforcement registry-lookup cost on Soroban; whether Rhinestone Registry attestation is actually validated at install time in current releases.
- **Freighter** — the Q4-2025 "Advanced auth … Soroban smart-wallet … progressive security" roadmap item.
- **`[T2]` Privy post-Stripe Stellar roadmap** — PYUSD-on-Stellar could promote Tier 3 support.
- **`[T2]` Fireblocks `PolicyRule` JSON schema + enum values** — OpenAPI spec needed; whether audit-log events are cryptographically chained/signed.
- **`[T2]` MetaMask Snaps** — exact audit-trigger capability list; per-Snap memory/CPU caps; cross-Snap RPC scope evolution; wallet-owned approval dialog primitive.
- **`[T2]` Tether WDK** — Stellar roadmap; whether `name@tether.me` has a public resolver; WDK participation in Open Wallet Standard.
- **`[T2]` Phantom MCP** — whether Phantom's account portal exposes server-side per-tx/per-period limits; whether OAuth `scopes` vocabulary is agent-configurable.

These feed `analysis/03-requirements.md` candidates queue and the stream-2 Stellar capability research.
