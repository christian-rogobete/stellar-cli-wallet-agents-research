# Coinbase Agentic Wallet (CDP)

**Status:** complete
**Researcher:** research agent (opus-4-7)
**Last updated:** 2026-04-18
**ID:** coinbase-agentic
**Category:** crypto-agent

---

## 1. What it is

Agentic Wallet is a hosted wallet product from Coinbase Developer Platform (CDP) layered on top of **CDP Wallets** (also called Server Wallet v2). Its pitch: "Agentic Wallet gives any AI agent a standalone wallet to hold and spend stablecoins, or trade for other tokens on Base. Built on Coinbase Developer Platform (CDP) infrastructure, agents can authenticate via email OTP, hold USDC, and send, trade, or pay for services without ever accessing private keys." [1] The underlying CDP Wallet infrastructure runs private-key operations inside **AWS Nitro Enclaves**; a `npx awal` CLI plus an "Agent Skills" library is the agent-facing surface, and an MCP server exposes operations as typed tools. [2][5][11]

## 2. Agent interface

The canonical agent surface is a CLI installed via `npx awal` plus skill packages installed with `npx skills add coinbase/agentic-wallet-skills`. [5] A minimum end-to-end sequence for a signed action:

1. `npx awal auth login <email>` — email-OTP authentication against CDP. [1][11]
2. `npx awal status` — confirm wallet exists and is funded. [1]
3. Skill invocation (LLM-driven) or direct CLI: `npx awal send 1 vitalik.eth` or `npx awal trade 5 usdc eth`. [1]
4. CDP API receives the request, authenticates it with a Wallet Secret (ECDSA P-256), forwards it into the Nitro Enclave over VSOCK, which decrypts the private key, signs, and returns the signed transaction. [3]

Parallel interfaces exist: a general-purpose `cdp` CLI, a CDP REST API (`api.cdp.coinbase.com/platform/v2/...`), and a CDP MCP server that "exposes every CDP operation as a typed tool" for AI agents. [9]

## 3. Custody model

Coinbase describes the product as "non-custodial" and "self-custody". [2][7][10] The technical implementation is:

- Private keys are generated and used inside a CDP-operated Trusted Execution Environment: "The TEE is hosted on AWS Nitro Enclaves, an isolated, secure compute environment." [3] Confirmed independently by AWS: "The core of the CDP Wallet API design is the verifiable application running within Nitro Enclaves." [6]
- "Account private keys are encrypted and decrypted exclusively within the enclave, and never leave the TEE. An encrypted version of the private keys are stored in CDP's database and can only be accessed by the developer with the corresponding Wallet Secret." [3]
- The Wallet Secret is "an asymmetric private key that conforms to ECDSA" on the secp256r1 curve, held by the developer and configured in the CDP Portal. [3]
- Enclave properties: "no persistent storage, no interactive access, and no external networking, ensuring that even a root or admin user cannot access or SSH into the TEE." [3]
- The agent never holds the signing key: "Private keys remain in secure Coinbase infrastructure, never exposed to the agent's prompt or LLM." [2]

The trust model is therefore: signing requires (a) the developer's Wallet Secret to authenticate the API call and (b) Coinbase-operated enclave infrastructure being alive, honest, and loaded with the advertised enclave image. Public-facing docs do not expose Nitro Enclave **attestation documents** to the developer; the published security page describes containment but not third-party-verifiable attestation. [3]

## 4. Policy and approval model

Policies are first-class and enforced server-side inside the enclave: "All policies are managed via API or SDK, and enforced at the enclave layer, even in the event of credential compromise." [7]

Two scopes: **project-level** (one per CDP project, applies to all accounts) and **account-level**. Project policies evaluate first, then account-level. [8][12]

Criteria types documented in the v2 REST reference [12]:

- `ethValue` — transaction value.
- `evmAddress` — recipient address (used for allowlists/denylists via `in` operator).
- `evmData` — ABI-aware function-and-argument matching.
- `netUSDChange` — USD-denominated value change, unit "cents" (e.g. `"changeCents": 10000`).
- `evmNetwork` — network scoping.
- `evmMessage`, `evmTypedDataField`, `evmTypedDataVerifyingContract` — message/EIP-712 signing restrictions.
- Solana equivalents: `solAddress`, `solValue`, `splAddress`, `splValue`, `mintAddress`, `solData`, `programId`, `solNetwork`, `solMessage`.

Operations that can be gated include `sendEvmTransaction`, `signEvmTransaction`, `prepareUserOperation`, `sendUserOperation`, and `signEndUser*` variants. [12] Rules are `accept`/`reject` with first-match stop semantics. [8]

There is **no explicit time-window criterion** and no built-in per-period rate limit in the public policy schema [12]; Agentic Wallet marketing mentions "Configurable caps per session and per transaction" [2] but the session-cap primitive is not exposed in the documented CDP policy schema — it appears to be Agentic-Wallet-specific and managed via the CDP Portal. KYT/OFAC screening is applied automatically as a compliance layer. [2]

The EVM Smart Account `spend-permissions` endpoint does express `allowance`, `period` (seconds), `start`/`end` timestamps, and a `spender` — an on-chain ERC-4337 primitive usable by developers but distinct from the off-chain Policy Engine. [13]

Policies are mutated via the CDP API with an API key scoped "Manage (modify policies)"; the agent has no authority to change them. [8]

## 5. Delegation model

Two patterns:

1. **Separate accounts per agent under one project**, each bound to an account-level policy. The developer holds the root Wallet Secret; the agent authenticates via OTP to its account. No key is shared — delegation is expressed as scope of the Wallet Secret plus attached policies. [8]
2. **ERC-4337 smart-account spend permissions** (`POST /platform/v2/evm/smart-accounts/{address}/spend-permissions`) granting a `spender` address an `allowance` of a given token per `period`, bounded by `start`/`end`. [13] This is on-chain delegation for Base/EVM; it does not exist on other chains.

Revocation is `DELETE /policies/{policyId}` or `revoke-spend-permission` on the smart-account path. [13] Propagation is immediate at the enclave layer (next request is evaluated against the updated policy set).

## 6. Relevant protocol coverage

- **Chains.** Agentic Wallet: **Base only** ("gasless trading on Base"). [1][4] The underlying Server Wallet v2 supports "EVM networks and Solana". [4] **No Stellar support, no Stellar roadmap announcement.** Verified against the CDP chain list and the Wallet Comparison page. [4]
- **x402.** Deep integration: pre-built skills `search-for-service`, `pay-for-service`, `monetize-service`; x402 Facilitator endpoints are exposed on the CDP API. [5][12] x402 runs on USDC on Base/EVM, not on Stellar (Coinbase's implementation).
- **Smart accounts.** ERC-4337 with spend permissions, batching, and CDP Paymaster on Base. [13]
- **Account types.** EOA, ERC-4337 smart account, Solana account.

## 7. Licence and adoption signals

- **Agentic Wallet service:** proprietary hosted service; no source available.
- **`coinbase/agentic-wallet-skills` GitHub repo:** MIT. [5] Contains only the skill manifests that wrap the `awal` CLI; the CLI itself is distributed via npm and is not source-available.
- **CDP AgentKit (`coinbase/agentkit`):** open source (Apache-2.0 per repo), framework-agnostic wallet tool library. [11]
- **Launch dates:** CDP Wallets (beta): April 2025 per [7]; Agentic Wallets: February 2026. [14]
- **Production users cited by Coinbase:** Questflow Labs, FereAI. [10][14]
- **Backer:** Coinbase (public, NASDAQ:COIN).
- **Adjacent deprecation:** "Server Wallet v1 (formerly known as Wallet API and MPC Wallet) is being deprecated as of Feb 2, 2026" — Coinbase has moved off MPC and onto pure TEE. [4]

## 8. Non-negotiable check

Against `analysis/00-context.md` §4:

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | **partial** | Coinbase markets "self-custody" [2][7]; in practice keys live in a Coinbase-operated enclave, signing requires Coinbase infrastructure to be alive and honest. Developer holds a Wallet Secret; user never holds the signing key. This is "enclave-resident, API-gated" custody, not user/host custody. [3] |
| N2 | Autonomous (no project-operated backend) | **no** | All signing flows through `api.cdp.coinbase.com`. No offline operation. [3][12] |
| N3 | No central server for keys/policy/history | **no** | Encrypted keys stored in "CDP's database" [3]; policies stored and enforced at CDP; "CDP Portal: Usage monitoring and agent management dashboard" holds history/telemetry. [2] |
| N4 | Open source, permissive licence | **partial** | Skill manifests (MIT) and AgentKit are OSS [5][11]; the wallet service, enclave image, `awal` CLI, and policy engine are proprietary. |
| N5 | JSON-default output | n/a | REST API is JSON; `awal` CLI output format not documented as JSON-first in primary sources. [12] |
| N6 | Testnet/mainnet parity | n/a | Base mainnet and Base Sepolia both supported via `evmNetwork`. [12] |

**Verdict:** fails N2 and N3 as a deployment model. Retain as a design reference for the policy-expression API surface.

## 9. What to adopt

- **Typed-criteria policy schema over free-form scripts.** `ethValue`, `evmAddress`, `evmData`, `netUSDChange`, `evmNetwork`, plus Solana equivalents are a clean enumeration: each criterion names the field, an operator (`<=`, `in`, ...), and a value [12]. Stellar-equivalent criteria would be `xlmAmount`, `assetCode+issuer`, `destination`, `networkPassphrase`, `operationType`, `hostFunction`, `contractAddress`, `sorobanAuthSubject`. Policy lives outside the agent-callable surface; a separate API-key scope is required to change it. (Threat model §6.1, T2/T4.)
- **Two-level policy scoping.** Project-level and account-level, with project evaluated first. [8] Maps cleanly to orchestrator (A2) plus sub-account pattern: the orchestrator sets a project-level floor that sub-agent accounts cannot loosen.
- **USD-denominated criterion (`netUSDChange`).** [12] A wallet that understands value in fiat-equivalents, not just asset units, defends against asset-unit confusion (T3). For Stellar this means resolving paths through known price sources at policy-evaluation time.
- **ABI/data-aware criterion (`evmData`).** [12] Policy can match on specific function selectors and argument values, not just amount and destination. Stellar-equivalent: match on Soroban host function, contract address, function name, and selected `ScVal` arguments in the auth entry.
- **EIP-712 typed-data and message-signing gates.** [12] Wallet does not blindly sign arbitrary messages or typed data; each class has its own criterion type. Stellar-equivalent: separate gates for SEP-10 challenge signing, SEP-45 entries, and arbitrary `signData`.
- **On-chain spend-permission primitive as a complement to off-chain policy.** [13] Delegation survives the wallet-service going down because it is enforced by the smart account contract. On Stellar this maps to smart-account (C-account) policy signers with narrow allowances — a design we should compare against `stellar-contracts/packages/accounts/`.
- **First-match evaluation with stop-semantics.** [8] Simple, explainable, auditable. Adopt instead of a weighted-score model.
- **Skill manifest pattern** (coinbase/agentic-wallet-skills [5]). Each skill is a small declarative bundle describing natural-language invocation plus underlying CLI call. Useful for exposing wallet operations to agent runtimes without coupling to one framework.

## 10. What to avoid

- **CDP-operated backend as the signing path.** Every signature requires `api.cdp.coinbase.com` to be reachable and uncompromised. Violates N2/N3 directly. [3][12] No offline mode, no user-host signing.
- **"Self-custody" terminology applied to enclave-resident keys.** The marketing claim "true self-custody" [2] depends on trusting Coinbase to run an honest enclave image with honest code and no secret side channels. The product is non-custodial relative to an individual Coinbase employee, but not relative to Coinbase as an organisation or its cloud provider. For our project, self-custody means "no third party can sign, period" (analysis/00-context.md N1); this product does not meet that bar.
- **No attestation surfaced to developers.** Public docs do not expose the Nitro Enclave attestation document (PCR measurements) to the caller [3]. A developer cannot cryptographically verify which enclave image handled their request. Any self-custodial design we borrow from must make attestation a first-class, user-verifiable artifact or not claim the property.
- **Policy history and telemetry on vendor servers.** [2] Fails N3 (privacy/no-history).
- **Chain lock-in to Base.** [4] Stellar parity is not on the roadmap. Reference value is the policy schema, not the chain.
- **KYT/OFAC as an inline, non-opt-out enforcement layer.** [2] Appropriate for a regulated US entity; inappropriate for a self-custodial primitive that should not make compliance decisions for the user.
- **Wallet Secret as a single bearer credential with "Manage policies" scope.** [3][8] A compromise of this credential loosens policy at the server layer. A self-custodial engine must bind policy-change authority to a separate approval channel (host-signed, hardware-backed, or multi-party).

## 11. Open questions

- Is the Nitro Enclave attestation document available to developers on request, or only to Coinbase internal auditors? Docs do not answer. [3]
- Does the Policy Engine support any time-window criterion (per-hour, per-day spend caps) beyond the smart-account-side `period`? Not exposed in the v2 criterion list. [12]
- What are Agentic Wallet "session caps" in technical terms — are they enforced in the enclave, in the API layer, or only in the `awal` CLI? Only documented in marketing copy. [2]
- Rate limits, quotas, and hard pricing ceilings for the free tier — `coinbase.com/developer-platform/pricing` returned 403 during research; one third-party source cites "5,000 operations per month free, $0.005/operation thereafter" [15] but this should be verified against the primary pricing page.
- Does Coinbase publish the enclave image's source or only a binary hash? No public reference found.
- Is there any Open Wallet Standard participation? Not mentioned in any CDP doc surveyed.

## 12. Sources

Primary sources only. Accessed 2026-04-18.

1. https://docs.cdp.coinbase.com/agentic-wallet/welcome — Agentic Wallet product overview, CLI surface, authentication flow.
2. https://www.coinbase.com/developer-platform/discover/launches/agentic-wallets — Launch announcement (Feb 2026); custody framing, guardrails list, architecture statement.
3. https://docs.cdp.coinbase.com/wallet-api/v2/introduction/security — Security architecture: AWS Nitro Enclaves, VSOCK, Wallet Secret, encrypted-key storage, enclave containment properties.
4. https://docs.cdp.coinbase.com/server-wallets/comparing-our-wallets — Chain support matrix (Agentic=Base only; Server=EVM+Solana; no Stellar anywhere).
5. https://github.com/coinbase/agentic-wallet-skills — MIT-licensed skill manifests; list of skills (authenticate, fund, send, trade, search-for-service, pay-for-service, monetize-service).
6. https://aws.amazon.com/blogs/web3/powering-programmable-crypto-wallets-at-coinbase-with-aws-nitro-enclaves/ — AWS confirmation of Nitro Enclave architecture, KMS via vsock, 10x throughput partnership.
7. https://www.coinbase.com/developer-platform/discover/launches/cdp-wallets-launch — CDP Wallets launch post; "All sensitive wallet interactions (like decrypting private keys) happen inside an AWS Nitro Enclave"; policy list.
8. https://docs.cdp.coinbase.com/server-wallets/v2/using-the-wallet-api/policies/overview — Policy model: project vs account scoping, first-match semantics, operation list.
9. https://docs.cdp.coinbase.com/get-started/tools/cdp-cli — CDP CLI taxonomy; MCP server "exposes every CDP operation as a typed tool".
10. https://www.coinbase.com/developer-platform/discover/launches/embedded-wallets-ga — Corroborating statement on TEE custody for Embedded Wallets (same security suite).
11. https://github.com/coinbase/agentkit — AgentKit framework-agnostic wallet toolkit (separate OSS project).
12. https://docs.cdp.coinbase.com/api-reference/v2/rest-api/policy-engine/policy-engine — Complete criterion-type list (EVM + Solana), operation list, example policies, `netUSDChange` cents-denominated units.
13. https://docs.cdp.coinbase.com/api-reference/v2/rest-api/evm-smart-accounts/create-a-spend-permission — ERC-4337 smart-account spend permission: `spender`, `allowance`, `period`, `start`/`end`.
14. https://www.coinbase.com/developer-platform/discover/launches/cdp-server-wallets-ga — GA of Server Wallets v2, confirms AWS Nitro Enclaves and v1 MPC deprecation.
15. https://docs.cdp.coinbase.com/server-wallets/v1/introduction/welcome — v1 Wallet API (MPC) deprecation notice dated Feb 2, 2026.

Blog citations (Decrypt, LeveX, Yahoo, etc.) were consulted during discovery but are not cited here per project source-priority rules.

## 13. Cross-links

- Compares directly to: **Turnkey** (same "hosted enclave-resident key" pattern, different tenant model), **Privy** (Shamir + Nitro hybrid), **Fireblocks** (MPC/HSM custodial).
- Contrasts with fully self-custodial candidates: **Trust Wallet Agent Kit**, **Meridian Pay**, **Safe + modules**, **MoonPay Agents + Open Wallet Standard**.
- Policy-schema lessons feed: `03-requirements.md` (policy primitives), `06-option-a-cli-first.md` §7 and `07-option-b-agent-native.md` §7 (auth-entry-level policy gates and typed tool boundary for policy mutation vs. spend operations).
- Open-question follow-ups: verify Nitro attestation availability; verify Agentic Wallet free-tier and rate limits against primary pricing page; confirm no Stellar roadmap statement has been made in any CDP changelog.
