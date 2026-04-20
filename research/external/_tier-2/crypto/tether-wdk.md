# Tether WDK / tether.wallet

**Status:** complete
**Researcher:** research-agent
**Last updated:** 2026-04-18
**ID:** tether-wdk
**Category:** crypto-agent

---

## 1. What it is

**Tether Wallet Development Kit (WDK)** is Tether's Apache-2.0 modular JavaScript SDK (Node.js + Bare runtime + React Native) for building self-custodial multi-chain wallets, open-sourced 17 October 2025 and integrated into the consumer `tether.wallet` app launched 14 April 2026 [1, 2, 3]. Architecture is an orchestrator (`@tetherto/wdk`) that registers per-chain wallet modules (`wdk-wallet-{btc,evm,evm-erc-4337,solana,spark,ton,ton-gasless,tron,tron-gasfree}`) and per-protocol modules (swap Velora, bridge USD₮0 via LayerZero, lending Aave V3, fiat MoonPay) under a single BIP-39 seed, with middleware hooks per-chain [4, 5]. Agent-facing surfaces ship separately: `@tetherto/wdk-mcp-toolkit` (35 MCP tools across 13 chains, v1.0.0-beta.1) [6] and `tetherto/wdk-agent-skills` (AgentSkills-spec `SKILL.md` file consumed by Claude/Cursor/Windsurf/OpenClaw via `npx skills add`) [7]. No CLI binary. No browser extension shipped by Tether (a community `wdk-starter-browser-extension` exists). `tether.wallet` is a closed-source iOS/Android app built on the kit; WDK itself is the open component [3].

## 2. Agent interface

Three surfaces, all derived from the same SDK:

1. **SDK (primary)** — `import WDK from '@tetherto/wdk'`; `new WDK(seed).registerWallet('ethereum', WalletManagerEvm, config).getAccount('ethereum', 0)` returns an `IWalletAccountWithProtocols` with `sendTransaction`, `transfer`, `getBalance`, `signMessage`, and `getSwapProtocol`/`getBridgeProtocol`/`getLendingProtocol` accessors [4, 5]. Stateless; no daemon.
2. **MCP server** (`@tetherto/wdk-mcp-toolkit`, Apache-2.0, 3 ★ as of 2026-04-18) — `WdkMcpServer` extends `@modelcontextprotocol/sdk`'s `McpServer`. Developer composes: `useWdk({seed}).registerWallet(...).usePricing().useIndexer({apiKey}).registerProtocol(...).registerTools([...WALLET_TOOLS, ...PRICING_TOOLS])`. Stdio transport by default; SSE-over-TLS mentioned for production [6, 8]. Tools declare MCP annotations `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint`; destructive tools use MCP elicitations for per-call human confirmation, disableable via `{ capabilities: { elicitation: false } }` [6, 8].
3. **Agent skill** (file-based, AgentSkills spec) — `npx skills add tetherto/wdk-agent-skills` installs `SKILL.md` + 12 reference files (chain deployments, per-module API). Agent reads the files and generates SDK code directly, not structured tool calls [7].

Minimum signed-action sequence (MCP path):

```
export WDK_SEED="twelve words..."
# launch wdk MCP server from client config (stdio)
# agent tool call:
#   tool: transfer
#   args: { chain: "plasma", token: "USDT", to: "0x...", amount: "10" }
# → MCP elicitation dialog shown to user; user approves
# → tx broadcast; returns { hash, fee }
```

## 3. Custody model

Single local BIP-39 seed, passed as a constructor argument or `WDK_SEED` env var; the library derives per-chain accounts via BIP-44 derivation paths [4]. No encrypted vault format is defined by WDK itself — the kit expects the embedding app to provide storage. The official React Native starter uses per-platform secure enclaves (iOS Keychain, Android KeyStore) via `@tetherto/wdk-react-native-core`'s Secret Manager with optional biometric unlock [9, 10]. Bare/Node samples pass the seed via environment variables; the MCP-toolkit setup wizard writes it into `.vscode/mcp.json` (gitignored) [8]. `WDK.dispose()` / `server.close()` clears seed material from memory [4, 6]. No hardware-wallet path in current modules. `WalletAccountReadOnly*` subclasses exist per chain for pubkey-only state queries without signing authority [11].

## 4. Policy and approval model

**None at the SDK layer.** The WDK core, wallet modules, and the MCP toolkit ship zero policy primitives: no amount caps, no counterparty allowlists, no rate limits, no time windows, no per-tx-type gating [4, 6, 8]. The sole approval primitive is the MCP elicitation shown to the user by the MCP client per destructive tool call — this is agent-runtime UX, not wallet-enforced policy, and the server-side flag can disable it. Scope control is limited to **selective tool registration**: `WALLET_READ_TOOLS` vs `WALLET_WRITE_TOOLS` exports let the embedder pre-filter what the agent can call, but a client that holds the write tool list has unbounded spending authority up to wallet balance [6, 8]. The `registerMiddleware(blockchain, async(account)=>...)` hook runs per-derived-account but is documented for "logging or failover protection," not policy enforcement, and does not intercept sign or broadcast calls [4, 5]. The agent skill ships prose "security guidance" (pre-transaction validation checklists, prompt-injection detection rules) that the LLM may or may not follow [7].

## 5. Delegation model

**No delegation primitive.** One seed produces N BIP-44 accounts (`getAccount('ethereum', 0..N)` or `getAccountByPath('ethereum', "1'/2/3")`), but all accounts share the same seed and no authority-scoping exists between them [4]. No capability tokens, no session keys, no sub-key expiry, no on-chain policy contract. The MCP toolkit's `useWdk({seed})` initialises the engine with one seed; there is no per-client or per-tool seed scope. Multiple agents wanting separate authority must run separate `WdkMcpServer` instances with separate seeds, and nothing in the kit binds one instance's scope relative to another's on-chain. Delegation of authority on an ERC-4337 account would come from the underlying smart-contract account (Safe modules etc.), not from WDK — the `wdk-wallet-evm-erc-4337` module uses Safe modules v0.3.0 [11, 12], so the Safe account's own delegation model applies, but WDK does not expose or wrap it.

## 6. Relevant protocol coverage

- **Chains with first-party modules:** Bitcoin (BIP-84 SegWit + Electrum), EVM (BIP-44 + EIP-1559), EVM ERC-4337 (Safe modules + Pimlico/Candide bundler/paymaster), Solana (Ed25519 + SPL), Spark (Lightning, key-tree, deposits/withdrawals), TON + TON gasless (Jettons + paymaster), Tron + Tron gas-free (TRC-20 + gasFreeProvider) [5, 11]. Community `@base58-io/wdk-wallet-cosmos` exists [13].
- **Stellar is absent** — not in any first-party module list, not in the agent-skill `chains.md` reference, not in the MCP toolkit's 13-chain enum [5, 6, 7]. `create-wdk-module` uses "stellar" only as the canonical example-name when scaffolding a *new* module [14] — a telling choice but not a supported chain.
- **x402 (EIP-3009 `TransferWithAuthorization`):** supported via community `@semanticio/wdk-wallet-evm-x402-facilitator` (Beta; Semantic Pay, not Tether); `WalletAccountEvm` satisfies the `ClientEvmSigner` interface directly so any WDK EVM wallet is a drop-in x402 client. Recommended chains are Plasma (`eip155:9745`) and Stable (`eip155:988`), both purpose-built for USD₮ with near-zero fees [15].
- **Gas-free transfer model:** three distinct mechanisms — (a) ERC-4337 paymaster on EVM (USDT paid as fee via Pimlico/Candide), (b) TON paymaster (`wdk-wallet-ton-gasless`), (c) Tron "GasFree" service provider (`wdk-wallet-tron-gasfree`) [5, 11]. All three require a third-party paymaster/bundler/gas-free-provider endpoint configured by the embedder — the fees are not literally free, they are paid in-asset and a service pays native-token gas in exchange.
- **`name@tether.me` identifier primitive:** a `tether.wallet` product feature, **not a WDK primitive**. There is no federation protocol, resolver API, CLI tool, or SDK module for `name@tether.me` anywhere in `tetherto/wdk*` or `tether-wdk-docs` [16, 17, 18]. Tether is the operator of the `tether.me` namespace; the mapping from username to on-chain address is centralised to Tether and unavailable to WDK callers programmatically [2, 16]. This is a UX layer on top of the proprietary app, not an open design element.

## 7. Licence and adoption signals

- **Licence:** Apache-2.0 across all `tetherto/wdk*` first-party repos (`wdk`, `wdk-docs`, `wdk-mcp-toolkit`, `wdk-agent-skills`, `wdk-wallet-*`, `wdk-protocol-*`, `wdk-indexer-http`) [1, 4, 6]
- **Source available:** yes (`tetherto` GitHub org, 130 public repos) [19]
- **Last meaningful commit:** `tetherto/wdk` pushed 9 April 2026 (v1.0.0-beta.7); `wdk-docs` 16 April 2026; `wdk-mcp-toolkit` 30 March 2026 (v1.0.0-beta.1); `wdk-agent-skills` 24 February 2026 [20]
- **GitHub stars:** `wdk` 171 ★; `wdk-docs` 18 ★; `wdk-mcp-toolkit` 3 ★; `wdk-agent-skills` 1 ★ (as of 2026-04-18) [20]
- **Production users:** `tether.wallet` (Tether's own app, launched April 2026); Rumble Wallet (80M Rumble users, integrated January 2026) [2, 21]; community showcase lists 4 projects (`wdk-starter-browser-extension`, `wdk-wallet-evm-x402-facilitator`, `x402-usdt0`, Novanet zkML Guardrails) [22]
- **Commercial backing:** Tether Operations Limited (world's largest stablecoin issuer); explicit partner program ("WDK Tech Contributors") inviting swap/DEX/lending module contributors [23]
- **Governance / standards posture:** Tether WDK is **not** listed among the 20 Open Wallet Standard contributor organisations [24, 25]. Conversely, OWS ships a Tether WDK adapter on its side ("adapters ship for viem, @solana/web3.js, and Tether WDK") — that is, OWS treats WDK as a downstream signer to wrap, not a peer spec [24, see `research/external/crypto/moonpay-agents-ows.md` §6 source 1]. WDK defines its own orchestrator and interface conventions rather than conforming to a cross-vendor standard.

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | Keys derived locally from a BIP-39 seed the embedder provides; never leaves the host in SDK or MCP-toolkit code [4, 6]. |
| N2 | Autonomous (no project-operated backend) | partial | **Core SDK**: yes, zero network dependency beyond the RPC endpoints the embedder configures. **Several modules depend on Tether- or vendor-operated services**: `wdk-indexer-http` requires an API key from `wdk-api.tether.io` for balance/transfer history [26]; fiat module depends on MoonPay; bridge module depends on LayerZero; ERC-4337 module depends on Pimlico/Candide; Tron gas-free depends on a `gasFreeProvider`. None of these are required for basic signing, but all of them are the documented happy path. |
| N3 | No central server for keys/policy/history | partial | Keys yes (self-custodial). Policy no (none exists to centralise). **History**: the canonical indexer is `wdk-api.tether.io` and all sample apps bake it in [26]. The MCP toolkit also ships a Bitfinex pricing client as a default capability (`usePricing()`) [6]. `name@tether.me` identifier resolution is centrally operated by Tether [2, 16]. |
| N4 | Open source, permissive licence | yes | Apache-2.0 across all first-party WDK repos [1, 4, 6]. |
| N5 | JSON-default output | n/a (SDK) / partial (MCP) | The SDK returns typed JS objects. The MCP toolkit returns MCP `structuredContent` JSON per tool (Zod-validated). No CLI to evaluate. |
| N6 | Testnet/mainnet parity | partial | Each wallet module accepts arbitrary `provider` / `host` / `network` values, so testnet is reachable, but the kit has no ambient network identifier — chain selection is a string passed to `registerWallet` and nothing structurally prevents a testnet RPC URL registered under the name `"ethereum"`. No required `--network` equivalent; **T10** is not structurally defended at the kit layer. |

## 9. What to adopt

Delta-focused against MoonPay OWS (primary comparison target per `research/external/_summary.md`). Most WDK agent-facing patterns are weaker versions of what OWS already provides — the legitimate deltas are few:

1. **MCP tool-annotation discipline** (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` on every tool) [6] — OWS does not specify MCP as a normative profile. Adopting these four annotations verbatim on every Stellar wallet MCP tool is a cheap win for agent-runtime UX and is the right shape regardless of policy location. Serves **A1-A3**.
2. **MCP elicitation as a structured per-call confirmation channel** [6, 8] — distinct from the agent's own UI; the client renders it. This addresses part of **T9** (approval-channel manipulation) for MCP-hosted clients that implement elicitation faithfully, giving us a wire-protocol anchor for the approval channel OWS §10 lacks. Note: **not sufficient** as the only approval primitive (it is disableable by capability negotiation), but a useful baseline.
3. **Read-only vs write tool arrays as a first-class export** (`WALLET_READ_TOOLS` vs `WALLET_WRITE_TOOLS`, plus per-category `*_READ_TOOLS` / `*_WRITE_TOOLS`) [6, 8] — a coarse but real scope-selection primitive. Our MCP surface should expose equivalent typed groupings so an embedding app can instantiate a read-only MCP server in one line. Serves **A6** and the read-path first-class requirement.
4. **`parseAmountToBaseUnits(amount, decimals)` utility with typed error codes** (`EMPTY_STRING`, `INVALID_FORMAT`, `NEGATIVE_AMOUNT`, `EXCESSIVE_PRECISION`, `INVALID_DECIMALS`, `SCIENTIFIC_NOTATION_PRECISION`) [6] — a deliberate guard against **T3** (hallucinated amount units). We should ship the Stellar equivalent for stroops ↔ XLM and for arbitrary-decimal Soroban tokens, with the same error-code surface.
5. **`MCPServer` extension pattern over subprocess wrapping** [6] — WDK's server extends `McpServer` directly and reuses its `registerTool`. OWS's MCP server is vendor territory; WDK shows the pattern is clean enough to standardise. Serves **A7**.
6. **Modular chain-at-a-time orchestrator with a single seed** (`registerWallet(name, ManagerClass, config)`) [4, 5] — a useful *shape* for our SDK/CLI where one Stellar wallet session can handle mainnet + testnet + a futurenet simultaneously under distinct registrations. Pair with a strict `--network` contract rather than the free-form strings WDK accepts.
7. **AgentSkills-spec `SKILL.md` + per-area reference files** [7] — for agent-runtime contexts that do not speak MCP (Claude projects, Cursor `.cursor/skills/`, Windsurf, OpenClaw). Ship our own `SKILL.md` alongside the MCP server, following the same spec, so a single artefact serves both transports.
8. **Default-tokens auto-registration** (`DEFAULT_TOKENS` registers USDT for supported chains without the embedder calling `registerToken`) [6] — our equivalent: auto-register common Stellar issuers (USDC Circle, EURC, yXLM) with home-domain-verified metadata so the agent does not receive arbitrary asset strings.

## 10. What to avoid

All of OWS §10's "avoid" bullets apply to WDK and **mostly apply more strongly** because WDK ships less. Specific WDK items:

1. **No policy engine at all** — not even the three declarative rules OWS defines (`allowed_chains`, `expires_at`, `allowed_typed_data_contracts`) [4, 6, 8]. The only scope lever is register-the-write-tool-or-don't. Fails **T2**, **T4**, **T6** by construction; **A1**'s "Must-have: enforceable spending limits" is unmet. Strictly worse than OWS for policy.
2. **Elicitation-only approval with opt-out flag** (`{ capabilities: { elicitation: false } }` disables confirmation entirely) [6] — any embedder misconfiguration removes the last safety net. Our design must make approval non-disableable for destructive tools.
3. **No delegation primitive, no session keys, no sub-authority scoping** [4, 5] — same-seed BIP-44 indexing is not authority scoping. Fails **T4** completely. For **A2** (multi-agent orchestrator) WDK is unusable — the operator must run entirely separate seeds and give up any shared-authority semantics.
4. **Seed as env var / mcp.json constructor arg is the documented happy path** [6, 8, 10] — plain `WDK_SEED` stored in `.vscode/mcp.json` or passed to `useWdk({seed})`. Fails **T1**: agent-process compromise = seed compromise. No OS-keyring-first pattern documented at the WDK-toolkit layer (the React Native starter does use Keychain/KeyStore, but that is starter code, not WDK library discipline).
5. **Indexer centralised at `wdk-api.tether.io` with a Tether-issued API key** [26] — the default history surface fails **N3** for balance/transfer queries. The MCP tool `getIndexerTokenBalance` uses this service by default. Our wallet must treat history providers as pluggable with a non-Tether default.
6. **`name@tether.me` is a closed, centralised naming system** [2, 16] — unusable as a design reference for Stellar federation. SEP-2 (federation) already gives us a decentralised equivalent anchored on `stellar.toml` home-domain; we must resist any temptation to build a single-namespace alias.
7. **"Gas-free" is rebranded paymaster delegation to third parties** (Pimlico / TON paymaster / Tron GasFree service) [5, 11, 15] — each is a central paymaster endpoint the user or embedder configures. On Stellar our native primitive is fee-bump transactions (CAP-15) and sponsored reserves (CAP-33), both decentralised and user-controllable. Don't adopt WDK's vendor-paymaster framing.
8. **No testnet/mainnet structural scoping** — chain name is a free string; registering a testnet `provider` under the name `"ethereum"` is allowed [4, 5]. Fails **T10**. Our wallet must require an explicit, enumerated network per operation.
9. **No hardware-signing path** — no Ledger/Trezor adapter in any first-party wdk-wallet-* module. **T1** is not mitigated for high-value attended operations (**A7**).
10. **Middleware hook is post-derivation-only** (`registerMiddleware(chain, async(account)=>...)` runs at account creation, not at sign/broadcast) [4, 5] — cannot intercept transactions. Do not confuse this shape with a policy-enforcement point.
11. **Stellar support must be built from scratch** [5, 14] — there is no wdk-wallet-stellar module; the `create-wdk-module wallet stellar` scaffold produces an empty template. Adopting WDK as a runtime would add a dependency layer on top of `stellar-cli` / our SDKs without giving us anything the tier-1 research does not already provide.

## 11. Open questions

- Does Tether have a public roadmap for Stellar support in WDK, given `create-wdk-module` uses Stellar as the example name? (Not found in `wdk-docs` or changelog through 2026-04-18.)
- Is the `name@tether.me` resolver a public service with an API, or strictly proprietary to the `tether.wallet` app? (Press coverage describes the behaviour; no public resolver endpoint documented.)
- Will Tether participate in Open Wallet Standard, or remain explicitly a separate kit with its own conventions? (Currently OWS ships a WDK adapter but WDK does not conform to OWS sub-specs.)
- What is the threat model Tether assumes for the `WDK_SEED` env-var delivery path? (No published threat model document; `.vscode/mcp.json` is flagged as gitignored but not encrypted at rest.)
- Does the planned ERC-4337 path integrate with Safe modules for policy (Allowance / Roles), or only for bundling? (Safe modules v0.3.0 referenced, but the wallet module exposes no Safe-module API.)
- Which, if any, of the WDK wallet modules currently ship conformance-test vectors, reproducible-build manifests, or release-signature verification? (Not found in skimmed repos.)

## 12. Sources

1. `https://tether.io/news/tether-champions-global-financial-freedom-infrastructure-with-open-source-release-of-its-wallet-development-kit-wdk/` — WDK open-source launch announcement, 17 October 2025 (accessed 2026-04-18).
2. `https://tether.io/news/tether-launches-tether-wallet-the-peoples-wallet-extending-its-global-financial-infrastructure-directly-to-billions-of-users-left-behind-by-the-traditional-financial-system/` — tether.wallet launch, 14 April 2026; `name@tether.me` identifier; launch chain list (2026-04-18).
3. `https://www.theblock.co/post/397358/tether-launches-self-custodial-wallet-supporting-usdt-bitcoin-and-tokenized-gold` — tether.wallet coverage; Rumble integration; chain list including Bitcoin Lightning (2026-04-18).
4. [`tether-wdk/src/wdk-manager.js`](https://github.com/tetherto/wdk/blob/main/src/wdk-manager.js) @ `40313a7efac412377320ed63809a1884012c7e86` (v1.0.0-beta.7) — `WDK` class, `registerWallet` / `registerProtocol` / `registerMiddleware` / `getAccount` / `getAccountByPath` / `dispose`; constructor takes `seed: string | Uint8Array`; no policy hooks (2026-04-18).
5. [`tether-wdk/README.md`](https://github.com/tetherto/wdk/blob/main/README.md) @ `40313a7e...` — module list, quickstart, orchestrator shape (2026-04-18).
6. [`tether-wdk-docs/ai/mcp-toolkit/api-reference.md`](https://github.com/tetherto/wdk-docs/blob/main/ai/mcp-toolkit/api-reference.md) @ `463a55bf682815283660547fae4955780485671a` — `WdkMcpServer`, 35 built-in tools, MCP annotations (`readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint`), elicitation for destructive tools, `parseAmountToBaseUnits` (2026-04-18).
7. [`tether-wdk-docs/ai/agent-skills.md`](https://github.com/tetherto/wdk-docs/blob/main/ai/agent-skills.md) and `skills/wdk/SKILL.md` @ `463a55bf...` — AgentSkills spec, `npx skills add tetherto/wdk-agent-skills`, 12 per-area reference files (2026-04-18).
8. [`tether-wdk-docs/ai/mcp-toolkit/get-started.md`](https://github.com/tetherto/wdk-docs/blob/main/ai/mcp-toolkit/get-started.md) and `configuration.md` @ `463a55bf...` — setup wizard generates `.vscode/mcp.json`; `useWdk({seed: process.env.WDK_SEED})`; `WALLET_READ_TOOLS` / `WALLET_WRITE_TOOLS` exports; security checklist (2026-04-18).
9. `https://github.com/tetherto/wdk-starter-react-native` — iOS Keychain / Android KeyStore via Secret Manager; biometric lock; BIP-39 12-word mnemonic (accessed 2026-04-18).
10. [`tether-wdk-docs/start-building/react-native-quickstart.md`](https://github.com/tetherto/wdk-docs/blob/main/start-building/react-native-quickstart.md) @ `463a55bf...` — `EXPO_PUBLIC_WDK_INDEXER_BASE_URL=https://wdk-api.tether.io`; provider URLs configured in `get-chains-config.ts` (2026-04-18).
11. `https://github.com/tetherto/wdk-wallet-evm` / `wdk-wallet-btc` / `wdk-wallet-ton` / `wdk-wallet-evm-erc-4337` — per-module READMEs; ReadOnly subclasses; ERC-4337 Safe modules v0.3.0 configuration (accessed 2026-04-18).
12. [`tether-wdk-docs/sdk/wallet-modules/wallet-evm-erc-4337/configuration.md`](https://github.com/tetherto/wdk-docs/blob/main/sdk/wallet-modules/wallet-evm-erc-4337/configuration.md) @ `463a55bf...` — `safeModulesVersion: '0.3.0'`; Pimlico bundler / paymaster; `paymasterToken` (2026-04-18).
13. [`tether-wdk-docs/overview/changelog.md`](https://github.com/tetherto/wdk-docs/blob/main/overview/changelog.md) @ `463a55bf...` — `@base58-io/wdk-wallet-cosmos` community module (2026-04-18).
14. [`tether-wdk-docs/tools/create-wdk-module.md`](https://github.com/tetherto/wdk-docs/blob/main/tools/create-wdk-module.md) @ `463a55bf...` — `npx @tetherto/create-wdk-module@latest wallet stellar` example scaffold (2026-04-18).
15. [`tether-wdk-docs/ai/x402.md`](https://github.com/tetherto/wdk-docs/blob/main/ai/x402.md) @ `463a55bf...` — x402 client / server / self-hosted facilitator; Plasma + Stable recommended chains; `WalletAccountEvm` is a drop-in `ClientEvmSigner`; community `@semanticio/wdk-wallet-evm-x402-facilitator` module (2026-04-18).
16. `https://www.coindesk.com/business/2026/04/14/tether-introduces-crypto-wallet-to-bring-stablecoin-and-bitcoin-payments-directly-to-users` — `name@tether.me` replaces wallet addresses (accessed 2026-04-18).
17. [`tether-wdk-docs/`](https://github.com/tetherto/wdk-docs) (full tree, grep for `tether\.me|federation|resolver`) — no matches @ `463a55bf...` (2026-04-18).
18. [`tether-wdk/src/`](https://github.com/tetherto/wdk/tree/main/src) (full tree) — no identifier-resolver code @ `40313a7e...` (2026-04-18).
19. `https://api.github.com/orgs/tetherto` — org metadata, 130 public repos (accessed 2026-04-18).
20. `https://api.github.com/repos/tetherto/{wdk,wdk-docs,wdk-mcp-toolkit,wdk-agent-skills}` — stars, `pushed_at`, license.spdx_id fields (accessed 2026-04-18).
21. `https://coinpedia.org/news/tether-launches-the-peoples-wallet-and-it-could-change-how-billions-use-crypto/` — Rumble Wallet first public WDK deployment, January 2026 (accessed 2026-04-18).
22. [`tether-wdk-docs/overview/showcase.md`](https://github.com/tetherto/wdk-docs/blob/main/overview/showcase.md) @ `463a55bf...` — community projects list (2026-04-18).
23. [`tether-wdk-docs/overview/partner-program.md`](https://github.com/tetherto/wdk-docs/blob/main/overview/partner-program.md) @ `463a55bf...` — Tech Contributor / Project Partner tracks (2026-04-18).
24. `https://www.prnewswire.com/news-releases/moonpay-open-sources-the-wallet-layer-for-the-agent-economy-302722116.html` — OWS 20-org contributor list (2026-04-18); Tether not listed.
25. `research/external/crypto/moonpay-agents-ows.md` §7 — explicit OWS contributor organisations list; Tether absent (2026-04-18).
26. [`tether-wdk-docs/tools/indexer-api/api-reference.md`](https://github.com/tetherto/wdk-docs/blob/main/tools/indexer-api/api-reference.md) and `https://wdk-api.tether.io/register` — `x-api-key` header against `wdk-api.tether.io`; key requested via Tether form (accessed 2026-04-18).

## 13. Cross-links

- Primary comparison: `research/external/crypto/moonpay-agents-ows.md` — OWS ships a stronger policy engine, capability-bound tokens, and the cross-vendor standard posture WDK lacks. WDK ships a cleaner MCP toolkit shape but no policy or delegation.
- `research/external/crypto/kraken-cli.md` §9 — MCP tool-annotation discipline (`readOnlyHint` / `destructiveHint`) also present in Kraken CLI MCP; WDK confirms the pattern.
- `research/external/crypto/trust-wallet-agent-kit.md` — another commercial self-custodial CLI/MCP kit; WDK delta is SDK-modular architecture, TWAK delta is per-invocation safety flags.
- `research/external/crypto/safe-modules.md` — WDK's ERC-4337 module reuses Safe modules v0.3.0 but does not wrap their policy primitives; the delta is "we ignore what Safe gives us."
- Relevant to `06-option-a-cli-first.md` §7 and `07-option-b-agent-native.md` §7: WDK provides zero guidance for Soroban auth entries or SEP surface; confirms that tier-1 candidates remain the canonical design references for Stellar-specific behaviour.
- Surfaces no new requirement candidates beyond what OWS already surfaces. Confirms the "tier-2 deferred" pre-research decision: WDK does not fill a tier-1 gap; it is a weaker sibling of MoonPay OWS with a better MCP surface but no policy, delegation, or Stellar support.
