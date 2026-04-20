# Phantom MCP server

**Status:** complete
**Researcher:** research-agent (Opus 4.7 1M)
**Last updated:** 2026-04-18
**ID:** phantom-mcp-server
**Category:** crypto-agent (tier 2)

---

## 1. What it is

`@phantom/mcp-server` is a ~40-line Node.js package that re-exports `@phantom/cli` and runs it with `--mcp` [S1 `packages/mcp-server/src/index.ts`, `bin.ts`]. All tools live in the CLI (`packages/cli/src/tools/`). The CLI authenticates to Phantom's hosted wallet service via OAuth (SSO or RFC 8628 device code) and routes every signing call through `api.phantom.app/v1/wallets`; Phantom's backend KMS signs. Solana is first-class, five EVM chains (Ethereum, Base, Polygon, Arbitrum, Monad) are peers, and Hyperliquid perpetuals ship as a separate tool group. Stellar is not supported.

## 2. Agent interface

MCP stdio only. `phantom-mcp mcp add` registers with the agent; on first call, a browser opens for OAuth login. Minimum signed action:

```
wallet_addresses
transfer_tokens { networkId: "solana:mainnet", to, amount, confirmed: false }  # → simulation preview
transfer_tokens { ..., confirmed: true }                                        # → { signature, ... }
```

**Full tool list (26 tools)** [S1 `packages/cli/src/tools/index.ts:41-75`]. **Auth:** `phantom_login`. **Wallet:** `wallet_status`, `wallet_addresses`, `wallet_balances`, `wallet_rebalance`. **Solana:** `solana_send` (`send_solana_transaction`), `solana_sign` (`sign_solana_message`). **EVM:** `evm_send`, `evm_sign`, `evm_sign-typed` (EIP-712), `evm_allowance`. **Tokens/swaps:** `transfer` (`transfer_tokens`), `buy` (`buy_token`, same-chain + cross-chain), `simulate`, `pay` (`pay_api_access`). **Perps (Hyperliquid):** `perps_markets`, `perps_account`, `perps_positions`, `perps_orders`, `perps_history`, `perps_open`, `perps_close`, `perps_cancel`, `perps_leverage`, `perps_transfer`, `perps_deposit`, `perps_withdraw`. Every tool carries MCP `readOnlyHint` / `destructiveHint` / `openWorldHint` annotations [S1 e.g. `transfer-tokens.ts:109-113`].

## 3. Custody model

**Hosted KMS, not self-custodial.** `PhantomClient` is instantiated with `walletType: "user-wallet"` and `apiBaseUrl: "https://api.phantom.app/v1/wallets"` [S1 `packages/cli/src/session/manager.ts:115-118, 426-481`; S2 `@phantom/client` dist/index.mjs:275]. The local `~/.phantom-mcp/session.json` (0o600, dir 0o700) holds an Ed25519 "stamper" keypair (SSO flow) or P-256 OIDC tokens (device-code flow) that authenticate API requests; the signing key never leaves Phantom. An attacker who exfiltrates the session file gets API-scope access to the hosted wallet — equivalent to a bearer token — but does not get the private key. Revocation is through Phantom's account portal. Category-equivalent to Coinbase CDP / Turnkey, not to TWAK Mode A or OWS.

## 4. Policy and approval model

Advisory only. Destructive tools default `confirmed: false`, which runs a server-side simulation and returns `{ status: "pending_confirmation", simulation: { expectedChanges, warnings, block? } }` [S1 `transfer-tokens.ts:80-107`, `send-solana-transaction.ts:31-99`]. The agent is instructed (in the tool description) to present the preview before retrying with `confirmed: true`. The flow is **not enforced**: "If the user wants to skip simulation and execute immediately, pass confirmed: true directly" [S1 `transfer-tokens.ts:100-101`]. No amount caps, allowlists, rate limits, per-period ceilings, or local policy file. Approval UX is wallet-owned only at the initial OAuth redirect; per-transaction approvals are rendered by the agent. Fails **T9** on every call after login.

## 5. Delegation model

None exposed. Single `walletId` per session; `derivationIndex` (HD path) is the only scoping vector [S1 tool schemas]. No subaccounts, scoped tokens, capability delegation, or on-chain policy. Fails **A2** and **A4**.

## 6. Relevant protocol coverage

- **Chains:** `solana:mainnet`/`devnet`; `eip155:1|8453|137|42161|143`; Hyperliquid Hypercore (perps). Bitcoin and Sui addresses are derived at wallet creation but no MCP tools expose them.
- **Simulation:** server-side via Phantom's simulator, returned inline in the first call of the two-step flow.
- **Hyperliquid perps:** 12 tools signing EIP-712 typed actions (chainId 42161); `perps_deposit` routes through Phantom's cross-chain swapper [S1 `PERPS.md:12-24`].
- **`pay_api_access`:** Phantom-proprietary CASH-token daily-quota payment — signs a server-returned unsigned Solana tx, stores the signature as a per-day bearer [S1 `pay-api-access.ts:22-69`]. Not x402.
- **Schemas:** Zod (via `incur`), CAIP-2 identifiers, dual-unit amounts (`"ui"` | `"base"`), typed `EthereumAddressSchema` / `SolanaAddressSchema` / `Base64Schema`.
- **Session:** auto re-auth on 401/403 with local file invalidation [S1 `session/manager.ts:173-190`].

## 7. Licence and adoption signals

- **Licence:** MIT — `Copyright (c) 2024 Phantom Technologies, Inc` [S3 `LICENSE`]. The "Proprietary" label in `npm view` reflects a blank `license` field in the subpackage; the repo LICENSE governs.
- **Source:** `github.com/phantom/phantom-connect-sdk` (23-package monorepo).
- **Last commit:** `56a222b` "chore: sync from internal", 2026-04-17 [S1].
- **Stars:** 134 [S4].
- **npm:** `@phantom/mcp-server@1.2.2` published 2026-04-17; 24 versions; actively developed (v0.1.x → v1.2.2 in ~4 months) [S1 `CHANGELOG.md`, S5].
- **Backing:** Phantom Technologies, Inc.

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | **no** | Keys in Phantom's hosted KMS; user holds an OAuth session. |
| N2 | Autonomous (no project backend) | **no** | All signing/addresses/simulation hit `api.phantom.app`. |
| N3 | No central server for keys/policy/history | **no** | All three live at Phantom. |
| N4 | Open source, permissive licence | yes | MIT [S3]. |
| N5 | JSON-default output | partial | MCP-native, but no uniform error envelope. |
| N6 | Testnet/mainnet parity | partial | CAIP-2 scoping exists, but `pay_api_access` hardcodes `SOLANA_MAINNET` [S1 `pay-api-access.ts:50-54`] and quotes default to mainnet. |

**Verdict: fails N1/N2/N3.** Not a deployment model. Reference for MCP tool-shape conventions only.

## 9. What to adopt

- **Two-step `confirmed: false → true` flow with server-side simulation** [S1 `transfer-tokens.ts:80-107`, `send-solana-transaction.ts:31-99`]. Makes simulation the default, submission the opt-in. For our wallet the simulation must run inside the trust boundary and the `confirmed` token must be an unforgeable nonce returned from the preview step, not a free boolean the agent can set directly. Serves **T2**, **T3**, **A4**.
- **MCP `readOnlyHint` / `destructiveHint` / `openWorldHint` annotations on every tool** [S1 `transfer-tokens.ts:109-113`, `sign-solana-message.ts:28-32`]. Standard MCP metadata enabling differentiated approval UX. Serves **T9** at the protocol layer.
- **Dual-unit amount schema (`amountUnit: "ui" | "base"`, default `"ui"`, explicit `decimals` for non-native)** [S1 `transfer-tokens.ts:54-73`]. Structural defence against **T3** amount-unit errors (stroops vs. XLM). Mirror for Stellar assets.
- **CAIP-2 on every tool + chain-namespaced tool names (`solana_*`, `evm_*`)** [S1 `transfer-tokens.ts:47-48`, `tools/index.ts:41-75`]. Avoids the TWAK duplicate-tool-name ambiguity and the network-confusion class (**T10**). Our equivalent: `stellar:pubnet` / `stellar:testnet` required on every tool; `stellar_*` namespace.
- **`simulate` as a first-class tool, not a flag** [S1 `simulate-transaction.ts`, registered alongside `send_*`]. Makes dry-run a peer of signing. Serves **A6**.
- **Auto re-authentication on 401/403 with session-file invalidation** [S1 `session/manager.ts:173-190`]. Clear retry semantics for **A1** / **A5**.
- **Tool descriptions as mini-protocols**: `transfer_tokens` description explicitly documents "TWO-STEP FLOW — always call this tool twice" with response shapes for both branches [S1 `transfer-tokens.ts:94-107`]. Pattern: the description is the only reliably-read contract with the agent.

## 10. What to avoid

- **Hosted-KMS custody with OAuth session as client credential.** Directly fails **N1/N2/N3** — the exact pattern our non-negotiables reject.
- **Advisory-only `confirmed` boolean.** The agent can set `confirmed: true` on first call and skip the preview entirely [S1 `transfer-tokens.ts:100-101`]. Under prompt injection it will. Fails **T2**.
- **No local policy layer.** No caps, allowlists, rate limits, per-period ceilings. Fails **T2**, **T4**, **T6**.
- **No delegation primitive.** Only HD `derivationIndex`. Fails **A2**, **A4**.
- **Base58 vs. base64 split for Solana transactions** (transaction base64; signature base58) [S1 `send-solana-transaction.ts:24-26`]. Solana ecosystem convention; do not port. Stellar XDR is base64 throughout — enforce a single encoding per artefact.
- **Simulation response shape modelled on Solana/EVM preflight** (blockhash freshness, ALT resolution, compute units / EIP-1559 gas). Soroban simulation returns `AuthorizationEntry` contexts, footprint, resource fees — a different object. Do not port the shape verbatim.
- **12 Hyperliquid perps tools** — EVM-specific DEX wrapped in EIP-712 typed actions (chainId 42161). Not generalisable to Stellar; skip entirely.
- **EIP-712 typed-data signing as a first-class MCP tool.** Stellar has no equivalent; the analogues are `stellar_sign_auth_entry` (SEP-43 / Soroban `SorobanAuthorizationEntry`) and `stellar_sign_sep10_challenge`.
- **`pay_api_access` as the paid-API primitive.** Phantom-proprietary (CASH-token + daily quota + signature-as-bearer). Not interoperable. Our path is x402 per SDF's x402-on-Stellar effort and/or SEP-10-anchored.
- **OAuth as ambient auth.** Serves hosted custody; irrelevant to self-custodial Stellar. Our auth is local keystore unlock, hardware-wallet plug, or passkey signer.
- **Agent-rendered approval after simulation.** Only the initial OAuth is wallet-owned; per-tx approvals are not. Fails **T9**.

## 11. Open questions

- Does Phantom's account portal expose per-tx or per-period limits server-side that constitute a policy layer outside the MCP surface? Unverified — would need an account.
- Is the OAuth `scopes` vocabulary agent-configurable, and does scope flow through to the API's signing authority?
- Does `pay_api_access` have any relationship to x402, or is it strictly Phantom-proprietary?
- Is the 401/403 auto-reauth loop rate-limited?

## 12. Sources

- **[S1]** Local shallow clone [`phantom-connect-sdk/`](https://github.com/phantom/phantom-connect-sdk) @ `56a222b3746e2d8565d04c4533ff3717b16baf2f` (2026-04-17). Upstream: `github.com/phantom/phantom-connect-sdk`. Accessed 2026-04-18. Primary source: `packages/mcp-server/{src/index.ts,src/bin.ts,README.md,PERPS.md,MCP_TEST.md,CHANGELOG.md}`; `packages/cli/{README.md,src/tools/*.ts,src/session/{manager.ts,storage.ts},src/auth/*.ts}`.
- **[S2]** `@phantom/client@2.0.1` tarball, `registry.npmjs.org/@phantom/client/-/client-2.0.1.tgz` (published 2026-04-17, accessed 2026-04-18). Confirms `walletType: "user-wallet" | "server-wallet"`, `signAndSendTransaction` routing, and the organisation/authenticator backend model.
- **[S3]** `github.com/phantom/phantom-connect-sdk/blob/main/LICENSE` — MIT, Copyright (c) 2024 Phantom Technologies, Inc. Accessed 2026-04-18.
- **[S4]** Repository metadata via `gh repo view phantom/phantom-connect-sdk`: public, MIT, 134 stars, pushed 2026-04-17. Accessed 2026-04-18.
- **[S5]** `npm view @phantom/mcp-server@1.2.2` and `npm view @phantom/cli@1.2.2`, accessed 2026-04-18.

## 13. Cross-links

- Compared to: `research/external/crypto/kraken-cli.md` (same "one binary, two transports" pattern via `phantom --mcp`); `trust-wallet-agent-kit.md` (Phantom's `<group>_<verb>` namespacing avoids TWAK's duplicate-tool-name failure); `moonpay-agents-ows.md` (OWS is the spec; Phantom has no OWS-equivalent capability-token or declarative policy); `stellar-ecosystem/stellar-mcp-server.md` (opposite custody failure mode — Stellar MCP puts `secretKey` in tool args; Phantom keeps the key at the vendor).
- Non-negotiable-failure cousin: `research/external/crypto/coinbase-agentic.md` and `turnkey.md` — vendor-KMS with client authenticator.
- Feeds requirements on: wallet-issued preview nonce for `confirmed` (T2, T3, T9); MCP tool annotations (T9); dual-unit amount schema (T3); CAIP-2 + per-chain namespacing (T10, T3); `simulate` as first-class tool (A6); Stellar-specific `signAuthEntry` / `signSep10Challenge` to replace EVM analogues; x402 over `pay_api_access` for paid-API.
