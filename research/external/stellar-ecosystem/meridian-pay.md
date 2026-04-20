# Meridian Pay (SDF smart wallet)

**Status:** complete
**Researcher:** research agent (Opus 4.7)
**Last updated:** 2026-04-18
**ID:** meridian-pay
**Category:** crypto-stellar

---

## 1. What it is

Meridian Pay is the Stellar Development Foundation's event-scoped smart-wallet demo, shipped at Meridian 2025 and extended afterwards as an open-source reference. A web front-end (no install) lets users create a Soroban contract-account authenticated by a device passkey; two back-end services (a Meridian-Pay-specific Node.js backend and the Stellar Disbursement Platform fork on the `meridian-pay` branch) handle fee sponsorship, batch wallet deployment, and platform-side recovery co-signing. The account contract, a router/multicall contract, an OpenZeppelin-based Merkle-airdrop contract, and an NFT contract make up the on-chain surface. SDF's public documentation explicitly states Meridian Pay "is intended to be used during Meridian 2025 and will not be maintained or supported long-term" [12.3].

## 2. Agent interface

None. Meridian Pay exposes only a browser UI and internal REST endpoints between the web app, the demo backend, the SDP backend, and the wallet-backend submitter. There is no public CLI, SDK, or MCP surface; the only scriptable entry points are admin scripts for Merkle-airdrop deployment (`npm run deploy-airdrop`, `generate-proofs`, `upload-proofs`) run by operators, not agents [12.2, 12.6]. A programmatic caller who wanted to drive Meridian Pay would have to impersonate its web client against the demo backend, and even then every state-changing call requires a WebAuthn passkey assertion from a platform authenticator — which cannot be produced by an unattended agent.

## 3. Custody model

Two tiers.

- **User passkey** (secp256r1 public key) is stored on-chain as the sole `DataKey::Signer` of the account contract; `__check_auth` verifies a WebAuthn assertion against it [12.4, `contracts/smart-wallet/src/lib.rs:20-98`].
- **Recovery Address** is stored on-chain as `DataKey::Recovery` and is the *only* caller permitted to call `rotate_signer`, which overwrites the passkey public key [12.4, `contracts/smart-wallet/src/lib.rs:42-71`]. In the demo the recovery address is a platform-held cosigner: wallet recovery requires both a server-held `RECOVERY_COSIGNER_PRIVATE_KEY` and a separate `cosignRecovery` call into the SDP backend, which countersigns the rotate_signer XDR [12.5, `apps/backend/src/api/embedded-wallets/use-cases/recover-wallet/index.ts:56-127`].

Submission and fees are fronted by the SDF-operated Wallet-Backend fee-bump service [12.1, 12.6]. Users pay no network fees; SDF pays them.

## 4. Policy and approval model

**There is no on-chain policy.** `__check_auth` receives `_auth_contexts: Vec<Context>` with a leading underscore and ignores it — any action the passkey signs is authorised [12.4, `contracts/smart-wallet/src/lib.rs:78-97`]. No spend caps, no destination allowlists, no time windows, no per-counterparty limits, no session-key scoping. Policy is expressed purely as "did a live human tap their fingerprint," which fails cleanly for T2 (prompt-injected transactions): an agent with scripted access to an unlocked passkey, or a user who habituates to tapping (T9), has no guardrails. This is the single most important gap for our threat model.

## 5. Delegation model

None. The account holds one passkey signer and one recovery address. There is no subaccount pattern, no session keys, no scoped auth entries, no ability to grant a sub-agent a bounded budget. The Merkle-distributor "pre-derive addresses from email salts" primitive is about *inbound* airdrops to not-yet-created wallets, not about outbound delegation, and does not generalise to agent delegation. Revocation of a lost or compromised passkey is the recovery flow (platform co-sign), which takes minutes to hours — unsuitable for A2 (orchestrator) or A3 (x402 service consumer).

## 6. Relevant protocol coverage

- Protocol 21 secp256r1 on-chain verification of WebAuthn signatures (required because the contract calls `env.crypto().secp256r1_verify` in `webauthn::verify`) [12.4, `contracts/smart-wallet/src/webauthn.rs`].
- `CustomAccountInterface` with `__check_auth` returning `Result<(), AccountContractError>` [12.4].
- OpenZeppelin `stellar-contracts` v0.3.0 used only for the airdrop + NFT side contracts (`stellar-merkle-distributor`, `stellar-fungible`, `stellar-non-fungible`) — **not** for the account contract itself [12.7, `contracts/Cargo.toml:8-13`; `contracts/airdrop/src/lib.rs:1-3` cites OZ `fungible-merkle-airdrop`].
- Multicall/router contract ported from Creit Tech's Stellar Router to bundle invocations under one passkey tap [12.7, `contracts/router/src/lib.rs:1-3`].
- SEP coverage: none in the contract; SDP handles SEP-style disbursement out-of-band.
- x402: **no relationship.** The strings `x402`, `402`, and `x402-mcp` appear in zero files across both repos (`grep -ri x402` returns no matches in `smart-wallet-demo-app` or `sdp-backend-meridian-pay`). The x402-on-Stellar + x402-MCP effort announced jointly with OpenZeppelin in early 2026 (see `analysis/00-context.md` §5.4) is a separate track; Meridian Pay's account contract is not the spending-policy enforcer that x402-MCP advertises.

## 7. Licence and adoption signals

- Licence (contract): Apache-2.0 (from the SDP repo that contains `contracts/smart-wallet/`) [12.8].
- Licence (demo monorepo): ISC declared in both root `package.json` and `apps/backend/package.json`; no separate `LICENSE` file at the demo repo root [12.2].
- Source available: yes (two repos: `stellar/smart-wallet-demo-app`, `stellar/stellar-disbursement-platform-backend` on branch `meridian-pay`) [12.2, 12.7].
- Last meaningful commit: 2026-04-17 on both repos (`smart-wallet-demo-app` @ `8f4bfdc`; `sdp-backend-meridian-pay` @ `0c84dff`). Active maintenance on SDP, light maintenance on the demo app.
- Known production users: one — Meridian 2025 event (~1,000 attendees, time-boxed). SDF explicitly scopes it out of long-term support [12.3].
- Commercial backing: SDF.
- Ecosystem traction: SDF-blessed canonical reference; cited in SDF's Smart Wallets documentation [12.9] but no downstream production deployments surfaced in search.

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | **partial** | User holds the passkey, but the recovery Address is the SDP platform cosigner [12.5] — SDF can rotate any user's signer with its own signature and a SDP co-sign. Pure self-custody is not achievable without running your own SDP fork + recovery cosigner. |
| N2 | Autonomous (no project-operated backend) | **no** | Wallet creation requires SDP; fee sponsorship requires wallet-backend; recovery requires SDP cosign. Three SDF-operated services, all mandatory [12.1, 12.5, 12.6]. |
| N3 | No central server for keys/policy/history | **no** | The demo backend stores user records (email, unique token, passkey metadata), and SDP stores the Merkle proofs, disbursement state, and mapping from email → contract address [12.1, 12.6]. |
| N4 | Open source, permissive licence | yes | Apache-2.0 for the account contract [12.8]; ISC for the demo monorepo [12.2]. |
| N5 | JSON-default output | n/a | No agent surface. |
| N6 | Testnet/mainnet parity | yes | Contracts and scripts accept `--network testnet|mainnet` [12.2, `contracts/airdrop/README.md`]. |

Meridian Pay is a useful **design reference**, not a deployment model. N1 partial + N2/N3 no remove it from direct adoption; it would require a significant re-architecture (remove SDP and wallet-backend dependencies, remove platform-held recovery cosigner, add on-chain policy) to satisfy the non-negotiables.

## 9. What to adopt

- **Deterministic address derivation by salt (email, user-id, or agent-name)** so parents can pre-compute a child's contract address before it exists. This is the one primitive that generalises to agent delegation: an orchestrator (A2) could deterministically derive a sub-agent's account address, pre-fund it via a Merkle-distributor contract with a per-child cap, and never hold the child's key. Source: `contracts/airdrop/src/lib.rs` + the email-as-salt deployer pattern described in [12.1].
- **Custom `__check_auth` for WebAuthn-only accounts** — thin (≈100 LOC) and auditable; worth copying verbatim as the passkey-signer baseline, then layering policy (A2/T4) on top [12.4].
- **Multicall/router contract pattern** to bundle several invocations under one human approval (A4, T9): reduces approval-channel habituation and matches our need for agent operations that must appear as one approval to the user [12.7, `contracts/router/src/lib.rs`].
- **Separation of submission from signing** via a fee-bump submitter (wallet-backend pattern) — but owned by the user's host, not by SDF. This satisfies N2 while keeping UX smooth.
- **OpenZeppelin `stellar-merkle-distributor` as a primitive for scoped inbound budgets** (A2, A3): the orchestrator or x402 buyer can pre-post a Merkle root with per-recipient caps; sub-agents claim without the root-signer staying online.

## 10. What to avoid

- **Single passkey as sole signer, with no `_auth_contexts` inspection** — fails T2 (any prompt-injected transaction is approved if the user taps), T4 (no scoped delegation), T6 (no rate limit). Our account contract MUST branch on `Context::Contract { contract, fn_name, args }` to enforce destination / amount / asset / frequency rules.
- **Platform-held recovery Address with a single `rotate_signer` call** [12.4, `contracts/smart-wallet/src/lib.rs:42-71`] — one corrupted cosigner (SDF, any custodian) replaces the user's signer at will. Fails N1. Use m-of-n guardians, time-locks, or the OpenZeppelin recovery module instead.
- **Mandatory SDF-operated submitter (wallet-backend) and wallet-creation service (SDP)** — fails N2/N3. The wallet must work end-to-end against a vanilla Stellar RPC + the user's host.
- **Web-app-only interface with per-action WebAuthn** — precludes A1 (unattended daemon), A5 (CI), A6 (read-mostly). Our surface must expose non-interactive signing paths (hardware wallet, OS-keyring ed25519, policy-scoped session keys) alongside passkeys.
- **SDP Merkle-distribution-as-primary-onboarding** — fine for an event, but ties wallet creation to an operator-initiated disbursement. The reference wallet must let a user (or agent) deploy their own contract from a seed they control.

## 11. Open questions

- Does the demo codebase anywhere call the contract's `rotate_signer` with a *user-held* recovery key in lieu of the platform cosigner, or is the platform cosigner the only supported path? The recovery use-case file only shows the platform-cosign flow [12.5].
- Is there a wasm-hash artifact reference we can pin for a published Meridian Pay account-contract deployment on mainnet, or was it redeployed per event?
- How does the fee-bump wallet-backend rate-limit per contract address? That mechanism would be a good lift-and-shift for our local policy engine, but its implementation lives in `stellar/wallet-backend`, which we have not cloned.
- Is SDF's x402-MCP effort planning to *reuse* this account contract, or will it ship its own policy-bearing account? No evidence either way in the current repos.

## 12. Sources

1. SDF blog post, "Building Meridian Pay: A Scalable Smart Wallet on Stellar." <https://stellar.org/blog/ecosystem/building-meridian-pay-smart-wallet-on-stellar> — accessed 2026-04-18. Overview of components, passkey flow, Merkle-distribution, multicall rationale.
2. `stellar/smart-wallet-demo-app` monorepo README and top-level config. [`smart-wallet-demo-app/README.md`](https://github.com/stellar/smart-wallet-demo-app/blob/main/README.md), [`smart-wallet-demo-app/package.json`](https://github.com/stellar/smart-wallet-demo-app/blob/main/package.json) @ `8f4bfdc`. Licence (ISC), Docker setup, SDP dependency.
3. SDF Meridian 2025 addendum / ToS. <https://stellar.org/meridian-addendum> — accessed 2026-04-18. States Meridian Pay "will not be maintained or supported long-term."
4. Account contract source. [`sdp-backend-meridian-pay/contracts/smart-wallet/src/lib.rs`](https://github.com/stellar/stellar-disbursement-platform-backend/blob/meridian-pay/contracts/smart-wallet/src/lib.rs), `.../src/webauthn.rs` @ `0c84dff`. `__check_auth`, `rotate_signer`, `CustomAccountInterface`, `DataKey::{Signer, Recovery}`.
5. Demo-backend recovery flow. [`smart-wallet-demo-app/apps/backend/src/api/embedded-wallets/use-cases/recover-wallet/index.ts:56-127`](https://github.com/stellar/smart-wallet-demo-app/blob/main/apps/backend/src/api/embedded-wallets/use-cases/recover-wallet/index.ts:56-127) @ `8f4bfdc`. Shows `RECOVERY_COSIGNER_PRIVATE_KEY` + `sdpEmbeddedWallets.cosignRecovery(...)` two-step.
6. Demo-backend wallet-backend + SDP interface modules. [`smart-wallet-demo-app/apps/backend/src/interfaces/wallet-backend/index.ts`](https://github.com/stellar/smart-wallet-demo-app/blob/main/apps/backend/src/interfaces/wallet-backend/index.ts), `.../sdp-embedded-wallets/index.ts` @ `8f4bfdc`. Confirms three-service architecture (demo backend + SDP + wallet-backend).
7. Contract workspace Cargo. [`smart-wallet-demo-app/contracts/Cargo.toml`](https://github.com/stellar/smart-wallet-demo-app/blob/main/contracts/Cargo.toml) @ `8f4bfdc`; `contracts/airdrop/src/lib.rs:1-3`, `contracts/router/src/lib.rs:1-3` @ `8f4bfdc`. OZ `stellar-contracts` v0.3.0 pin, Creit Tech Router attribution.
8. SDP repository licence. [`sdp-backend-meridian-pay/LICENSE`](https://github.com/stellar/stellar-disbursement-platform-backend/blob/meridian-pay/LICENSE) @ `0c84dff`. Apache-2.0.
9. Stellar docs, "Smart wallets." <https://developers.stellar.org/docs/build/guides/contract-accounts/smart-wallets> — accessed 2026-04-18. Confirms `__check_auth` + secp256r1 + passkey pattern; lists Passkey Kit as the primary SDK.

## 13. Cross-links

- **Counterweight to passkey-kit research** (`research/external/stellar-ecosystem/passkey-kit.md`, pending) — passkey-kit provides the client-side SDK; Meridian Pay shows one way to wire a backend around it. We can consume the SDK without adopting the backend.
- **Counterweight to stellar-contracts accounts** (`research/external/stellar-ecosystem/stellar-contracts.md`, pending) — OZ's `packages/accounts` offers composable signers and policies; Meridian Pay deliberately did **not** use it and ships a single-signer account. Our design record should state why we follow OZ, not Meridian Pay, on the account side.
- **x402 effort** (`analysis/00-context.md` §5.4) — Meridian Pay is not the x402 spending-policy layer; that layer does not yet exist publicly. Positioning: Meridian Pay is SDF's consumer-wallet demo; our RFP is for the agent-primitive SDF has not yet shipped.
- **Requirement hooks**: N1/N2/N3 failures feed directly into `03-requirements.md` (host-only custody, no mandatory platform services, policy as on-chain first-class citizen).
