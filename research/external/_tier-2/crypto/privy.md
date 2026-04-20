# Privy

**Status:** complete
**Researcher:** research-agent (opus-4.7)
**Last updated:** 2026-04-18
**ID:** privy
**Category:** crypto-agent

---

## 1. What it is

Hosted embedded-wallet + auth platform (Horkos, Inc.; acquired by Stripe 2025-06-11 [14]) provisioning wallets inside AWS Nitro Enclaves. Client SDKs: React / React Native / iOS / Android / Unity / Flutter. Server SDKs: Node / Python / Java / Go / Rust [1][2]. Two framings: **embedded wallets** (user-centric login via email/SMS/social/passkey) and **server wallets** (app-centric, controlled by authorization keys) [2][5]. 2025-2026 focus is "programmable custody" for agent-operated fleets [3][8].

## 2. Agent interface

REST at `api.privy.io` + SDKs [1]. Agent surfaces: Claude/Cursor **Agent Skill** (`privy-agentic-wallets-skill`, MIT, 10★ [4]); **docs MCP** at `docs.privy.io/mcp` [6]; **wallet-action MCP** (`privy-io/privy-mcp-server`, MIT [4][7]) with create-wallet, send-tx, sign-message/typed-data/raw-hash, policy management, key quorums. Minimum Stellar sequence: auth with `PRIVY_APP_ID`+`PRIVY_APP_SECRET`; create wallet `chain_type:"stellar"`; `raw_sign` a 32-byte tx hash [12]; assemble XDR client-side via `@stellar/stellar-sdk`; submit via the app's own Horizon/RPC.

## 3. Custody model

Three configurations; **Privy always holds one share and runs the enclave**.

- **Embedded, device-default.** 2-of-2 Shamir: device share + Privy-held auth share; reconstitution inside the Nitro Enclave, sign, wipe [9][10].
- **Embedded, delegated.** User consent re-provisions the wallet to a server-side TEE as a "second device" so the app backend signs without the user present [11].
- **Server wallet.** App-created; non-TEE share controlled by app-held **authorization keys** (P-256); every request carries `privy-authorization-signature`; optional **key quorum** (m-of-n) [13][15][16].

## 4. Policy and approval model

TEE-enforced JSON rules `{chain_type, rules[{method, conditions[{field_source, field, operator, value}], action}]}`, `action ∈ {ALLOW, DENY}` [17][19]. Chain-aware parsing only for `chain_type ∈ {ethereum, solana, tron, sui}` covering EVM transaction/calldata/typed-data, Solana program instructions, Tron and Sui variants. **No `stellar` chain_type** — Stellar goes through `raw_sign` on a hash; policy cannot assert on destination, asset, memo, op, or Soroban auth entry [17][12]. Policies carry `owner_id`; mutation itself requires authorization signatures [16]. No time-window, per-period, or rate-limit operator.

## 5. Delegation model

**Delegated actions** — user grants app signing permission; `revokeWallets()` revokes on next request [11][18]. **Key quorums** — m-of-n authorization keys on a server wallet, enforced at the key layer inside the TEE [15]. Authority is credential + policy, not a key share; revocation flows through Privy.

## 6. Relevant protocol coverage

- **Tier 3 (full parse + policy):** EVM, Solana [20].
- **Tier 2 (HD index 0, key export, `raw_sign`, no policy parsing):** Bitcoin, Cosmos, **Stellar**, Spark, Sui, Tron, Near, Ton, Starknet, Aptos, Movement [20].
- **Tier 1:** other Ed25519/secp256k1 chains, curve-level [20].
- **Stellar:** Ed25519, address derivation, HD index 0, `raw_sign`, key export. No Soroban, no SEPs, no policy parsing [12][20].
- **Smart wallets (EVM-only):** ERC-4337 over Kernel/Safe/LightAccount/Biconomy/Thirdweb/Coinbase Smart Wallet [brave-1].

## 7. Licence and adoption signals

- **Licence:** proprietary SaaS; SDKs + `privy-mcp-server` + `privy-agentic-wallets-skill` MIT/Apache-2.0; core service closed; not self-hostable [4][21].
- **Last commits:** `rust-sdk` 2026-04-16; `privy-agentic-wallets-skill` 2026-03-13; `privy-mcp-server` 2026-02-09 [4].
- **Stars:** `shamir-secret-sharing` 240; `privy-ios` 19; `privy-agentic-wallets-skill` 10 [4].
- **Spec adherence:** no OWS, no SEP; Global Wallets uses OAuth cross-app sharing [22].
- **Production users:** BankrBot, Hyperliquid, Vector, Blackbird; 120M+ accounts across 2,000+ teams [brave-1].
- **Backing:** Stripe (2025-06-11) [14]; prior $41.3M (Ribbit, Sequoia, Paradigm, Coinbase Ventures, Electric) [15].
- **Pricing:** Core free (0-499 MAU, 50k sigs); Scale $299/mo; Growth $499/mo; Enterprise from $0.001/sig [23].

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | **partial / no** | Default embedded mode has a real client-held share; delegated and server-wallet modes fully remove user-side custody. Privy always holds the TEE share [9][10][11]. |
| N2 | Autonomous | **no** | Every signature requires `api.privy.io` + a running Privy enclave [2][10]. |
| N3 | No central server | **no** | Privy holds the auth share, runs policy in-TEE, stores activity centrally [9][17]. |
| N4 | Open source, permissive | **partial** | SDKs MIT/Apache-2.0; core service closed [4][21]. |
| N5 | JSON-default | yes | REST/JSON [17]. |
| N6 | Testnet/mainnet parity | n/a | Chain-agnostic at signing layer [20]. |

Fails N1-N3 by construction. Design-reference only.

## 9. What to adopt

- **2-share device+server reconstitution.** User holds something real, unlike Turnkey's pure enclave [9][10]. Serves **A4**; better addresses **T1** than pure-TEE.
- **Delegation as TEE-re-provisioning, not a key share.** Distinct consent, not a time window on an existing key [11]. Serves **A2**, **A4**; addresses **T4**.
- **Authorization key separate from signing key.** `privy-authorization-signature` + quorums force a second proof per request [13][15][16]. Addresses **T1**, **T8**; serves **A7**.
- **Twin agent surfaces — Skill + wallet-action MCP.** Claude/Cursor Skill for the coding agent, MCP for the runtime agent [4][6][7]. Serves **A7**, **A1**.
- **`owner_id` on every policy.** Mutation requires owner's signature [16]; our local policy files should mirror this against host-compromise rewrites.

## 10. What to avoid

- **"Self-custodial" with hidden second custodian.** Turnkey's framing is pure branding; Privy has a real device share in *one* mode that disappears on delegation. Fails **N1** in delegated/server modes [11].
- **Closed policy engine bound to Privy liveness, 4 chains, no Stellar parsing.** Privy preferable to Turnkey on pricing; Turnkey preferable on grammar. Both fail **N2/N3** [17].
- **`raw_sign`-only Stellar.** No Soroban / memo / destination / SEP — identical to Turnkey [12][20].
- **Habituated delegation.** User may not track the 2-of-2-to-server-signed move on consent; **T9** [11].
- **No self-host path.** Turnkey ships AGPL QuorumOS; Privy nothing analogous [21].
- **Stripe concentration.** Post-acquisition positioning risk for a Stellar reference wallet [14].

## 11. Open questions

- Multi-device share-set topology after a second device.
- Any hash-prefix distinguisher (Stellar tx vs. Soroban auth-entry preimage) exposed to `raw_sign` policy on enterprise tier.
- Post-Stripe roadmap — does PYUSD on Stellar promote it to Tier 3?
- Recovery flow when the user loses both email/social auth *and* device share.
- Whether docs MCP and wallet-action MCP are ever co-deployed (**T8** isolation).

## 12. Sources

1. Privy docs homepage. `https://docs.privy.io/` (accessed 2026-04-18). SDK surface list.
2. Privy docs — Wallets overview. `https://docs.privy.io/wallets/overview` (accessed 2026-04-18).
3. Privy — Powering programmable wallets with low-level key management. `https://privy.io/blog/powering-programmable-wallets-with-low-level-key-management` (accessed 2026-04-18). TEE + sharding + in-enclave policy.
4. Privy GitHub org. `https://github.com/privy-io` (accessed 2026-04-18). Repo inventory, licences, stars, commits.
5. Privy docs — Server wallets. `https://docs.privy.io/guide/overview-server-wallets` (accessed 2026-04-18).
6. Privy docs — Build with AI tools. `https://docs.privy.io/basics/get-started/using-llms` (accessed 2026-04-18). Docs MCP; llms.txt; Agent Skills.
7. `privy-io/privy-mcp-server` listing. `https://lobehub.com/mcp/privy-io-privy-mcp-server` (accessed 2026-04-18). Tool list.
8. Privy — BankrBot case study. `https://privy.io/blog/bankrbot-case-study` (accessed 2026-04-18). Agent delegation on X.
9. Privy docs — Security architecture. `https://docs.privy.io/security/wallet-infrastructure/architecture` (accessed 2026-04-18). 2-of-2 shares; TEE reconstitution.
10. Privy — How embedded wallets work. `https://privy.io/blog/how-privy-embedded-wallets-work` (accessed 2026-04-18). Signing pipeline; AWS Nitro Enclave.
11. Privy — Introducing Delegated Actions. `https://privy.io/blog/delegated-actions-launch` (accessed 2026-04-18). Delegation as enclave re-provisioning; revocation.
12. Privy docs — Using Tier 2 chains. `https://docs.privy.io/recipes/use-tier-2` (accessed 2026-04-18). Stellar hash-signing with `@stellar/stellar-sdk`.
13. Privy docs — Security checklist. `https://docs.privy.io/security/implementation-guide/security-checklist` (accessed 2026-04-18). Authorization-key separation.
14. Privy — Stripe acquisition blog + Bloomberg. `https://privy.io/blog/announcing-our-acquisition-by-stripe`; `https://www.bloomberg.com/news/articles/2025-06-11/payment-company-stripe-to-acquire-crypto-wallet-provider-privy` (accessed 2026-04-18).
15. Privy docs — Signing with key quorums. `https://docs.privy.io/controls/key-quorum/sign` (accessed 2026-04-18). m-of-n threshold.
16. Privy docs — Authorization signatures. `https://docs.privy.io/api-reference/authorization-signatures` (accessed 2026-04-18). `owner_id`; mutation signatures.
17. Privy docs — Policies overview. `https://docs.privy.io/controls/policies/overview` (accessed 2026-04-18). `chain_type ∈ {ethereum, solana, tron, sui}`.
18. Privy docs — Revoking delegated wallets. `https://docs.privy.io/guide/react/wallets/embedded/delegated-actions/revoke` (accessed 2026-04-18).
19. Privy docs — Create policy API. `https://docs.privy.io/api-reference/policies/create` (accessed 2026-04-18).
20. Privy docs — Chain support. `https://docs.privy.io/wallets/overview/chains` (accessed 2026-04-18). Three-tier map; Stellar = Tier 2.
21. Privy — Wallets and architectures, part 2. `https://privy.io/blog/embedded-wallet-architecture-breakdown` (accessed 2026-04-18). TEE + SSS trade-offs.
22. Privy docs — Global Wallets overview. `https://docs.privy.io/wallets/global-wallets/overview` (accessed 2026-04-18). OAuth-provider cross-app.
23. Privy pricing. `https://www.privy.io/pricing` (accessed 2026-04-18). Core / Scale / Growth / Enterprise.

brave-1: Brave Web Search, "Privy embedded wallet agent delegation 2025" (accessed 2026-04-18). ERC-4337 and ERC-7715/7579/7555 coverage (docs snippet).

## 13. Cross-links

- Direct comparator: **Turnkey** (`research/external/crypto/turnkey.md`). Same hosted-enclave category; different share topology and developer posture. Deltas in §3/§9/§10.
- Indirect comparators: **Coinbase Agentic / CDP** (same vendor-enclave policy pattern); **MoonPay OWS** (spec-based alternative that passes where Privy fails on N1-N3).
- Contributes to requirements: per-resource `owner_id` with mutation-signed-by-owner (**T1/T8**); delegation-as-re-provisioning not key-sharing (**T4**); twin agent surfaces — Skill for the coding agent, MCP for the runtime agent (**A7**, **A1**); explicit multi-share custody labelling so "self-custodial" does not drift (N1).
