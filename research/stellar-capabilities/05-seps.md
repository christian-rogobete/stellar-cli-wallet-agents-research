# Stellar Ecosystem Proposals (SEPs) relevant to an Agent Wallet

**Status:** complete
**Last updated:** 2026-04-18
**ID:** 05-seps

---

## 1. What it is

This file is a **relevance-filtered index** of Stellar Ecosystem Proposals (SEPs) that a self-custodial AI-agent wallet plausibly touches. It is not a SEP encyclopaedia; SEPs unrelated to wallet, counterparty-identity, anchor-service, authentication, key-derivation, or Soroban-token concerns are deliberately out of scope.

Each SEP is summarised in the table in §2, and the SEPs that carry load for the agent-wallet decision get a short prose section afterwards describing **how a programmatic, possibly unattended caller uses or bypasses it**, which is the axis a conventional wallet review rarely covers.

SEP numbering and status in this file reflect the local clone of the `stellar-protocol` repository at short SHA `44f5f5d` ([`stellar-protocol/ecosystem/`](https://github.com/stellar/stellar-protocol/tree/master/ecosystem)). Where the "Status" field below disagrees with a newer upstream state, the local pin is authoritative for this analysis record.

## 2. Components — SEP summary table

| SEP | Title | Status | Actors | Threats mitigated / created | Non-negotiables touched | Agent-specific note |
|---|---|---|---|---|---|---|
| SEP-01 | Stellar Info File (`stellar.toml`) | Active (v2.7.0) [1] | A1, A3, A4, A5 | Mitigates T5; reads create T7 surface | N1, N3, N5 | Counterparty-identity foundation; the wallet resolves home domain → toml → signing key / endpoints. Must be cached with TTL and pinned-cert friendly for unattended use. |
| SEP-05 | Key Derivation Methods (BIP-39 / SLIP-10, path `m/44'/148'/x'`) | Final [2] | A2, A7 | Mitigates T1 (scoped subaccount keys); constrains T4 | N1 | Base layer for subaccount derivation per `research/brain-dump/requirements.md` bullet 21; agent must never expose mnemonic over the tool boundary. Derivation is local-only. |
| SEP-06 | Deposit and Withdrawal API (programmatic) | Active; interactive components deprecated in favour of SEP-24 [3] | A3 (indirect), A7 | Mitigates nothing directly; exposes T5, T10 | N1, N2, N3 | The only fully-programmatic anchor path. Depends on SEP-10, SEP-12, and optionally SEP-38. Many anchors require KYC that cannot be completed unattended — treat as semi-attended. |
| SEP-07 | URI Scheme (`web+stellar:`) for delegated signing | Active (v2.1.0) [4] | A4, A7 | Mitigates T9 (wallet-owned approval UI) | N1, N3 | The primary hand-off from an agent or third-party site to a signing wallet. For A4 (user-facing assistant with human approval) this is the approval channel. Wallet MUST treat URIs as untrusted input. |
| SEP-09 | Standard KYC Fields | Active (v1.17.0) [5] | A3 | n/a | n/a | Vocabulary only; relevant because SEP-12 / SEP-06 / SEP-31 carry SEP-9 field names. No direct agent behaviour. |
| SEP-10 | Stellar Web Authentication (classic `G` / muxed `M` accounts) | Active (v3.4.1) [6] | A1, A3, A5 | Mitigates T5 (identity-anchored auth); creates T9 surface (JWT handling) | N1, N2, N3 | Core prerequisite for almost every paid or KYC-gated service. Agent can run the challenge-response unattended because the challenge is a zero-sequence transaction with bounded semantics — see §4. |
| SEP-11 | Txrep (human-readable transaction format) | Active (v1.1.0) [7] | A4, A7 | Mitigates T3, T9 (preview renderable in human terms) | N5 | Useful as the deterministic preview format before signing. Referenced by SEP-07's `replace` parameter. Not a transport; a rendering. |
| SEP-12 | KYC API | Active (v1.15.0) [8] | A3, A7 | Mitigates nothing directly; creates T5 surface (identity transported) | N1, N3 | Supporting flow for SEP-6 / SEP-24 / SEP-31. Data is PII; wallet MUST NOT persist SEP-12 payloads server-side (see N3). |
| SEP-24 | Hosted Deposit and Withdrawal (interactive) | Active (v3.8.0) [9] | A4, A7 | Mitigates T9 when the interactive URL is opened in a wallet-owned window | N1, N3 | **By design no purely-agent path.** An unattended A1 cannot complete SEP-24 flows. The wallet's role is to authenticate with SEP-10/-45 and hand off the interactive URL to the human via companion UI. |
| SEP-29 | Account Memo Requirements | Active (v0.5.0) [10] | A1, A2, A3, A4 | Mitigates T5, T6 (wrong-memo loss on custodial destinations) | N5, N6 | Structural check before signing any payment/merge-op transaction. Wallet MUST refuse to sign a memo-less payment to a `config.memo_required=1` account, regardless of agent request. |
| SEP-30 | Account Recovery (multi-party) | **Draft** (v0.8.1) [11] | A7 | Mitigates T1 at the cost of T4 (introduces recovery signers) | N1 (trade-off: N1 vs. practical recoverability) | Recovery is a human, attended flow. Depends on SEP-10. Adds N recovery signers to the account, each operated by a separate recovery service. Non-trivial interaction with policy and thresholds. |
| SEP-31 | Cross-Border Payments API | Active (v3.1.0) [12] | A3 (if at all), A7 | n/a for agent wallet directly | N1, N3 | Anchor-to-anchor protocol, not client-to-anchor. Our wallet is a client; SEP-31 typically belongs to the anchor side. Listed for completeness; candidate for out-of-scope (§7.1). |
| SEP-38 | Anchor RFQ API (quotes) | Draft (v2.5.0) [13] | A3, A4 | Mitigates T3 (explicit quote id) | N5 | Price discovery for cross-asset deposit/withdraw or swap. Firm quotes are ID-referenced and time-bound; perfect for a deterministic agent flow. Auth via SEP-10 or SEP-45. |
| SEP-41 | Soroban Token Interface | Draft (v0.4.1) [14] | A1, A2, A3 | Mitigates T3 (typed `i128`, explicit decimals); creates T4 (approvals) | N1, N5 | Canonical ERC-20-analogue for Soroban. Wallet MUST understand `transfer`, `approve(..., live_until_ledger)`, `balance`, `allowance`. `approve` with a ledger lifetime is the Soroban equivalent of unlimited-allowance risk. Load-bearing for `04-smart-accounts.md` and `07-dex-amm.md`. |
| SEP-43 | Standard Web Wallet API Interface | Draft (v1.2.1) [15] | A4, A7 | Mitigates T9 (typed approval surface) | N5 | The canonical wallet-kit shape: `getAddress`, `signTransaction`, `signAuthEntry`, `signMessage`, `getNetwork`. Our agent-facing surface MUST be a superset or expose a SEP-43 adapter for interop with `stellar-wallets-kit`, Freighter, Lab, Blend, etc. |
| SEP-45 | Stellar Web Authentication for Contract Accounts | Draft (v0.1.1) [16] | A1, A3, A5 (where account is `C...`) | Mitigates T5; creates T3 surface (auth-entry args) | N1, N2, N3 | The **contract-account twin of SEP-10**. If the wallet uses a smart account (`C...`) as its primary account, SEP-10 does not apply — SEP-45 does. Uses Soroban auth entries on a `web_auth_verify(...)` contract rather than a zero-sequence transaction. Services must implement both if they accept both `G` and `C` accounts. |
| SEP-46 | Contract Meta | Active (v1.0.0) [17] | A1 | Mitigates T5 (build/version identity of a called contract) | N5 | Custom Wasm section where contracts declare metadata (source, build, `sep` list). Read by the wallet to identify what contract the agent is about to call. Dependency for SEP-47. |
| SEP-47 | Contract Interface Discovery | Draft (v0.1.0) [18] | A1 | Mitigates T3 (structural "does this contract implement SEP-41?") | N5 | `sep=41,40` meta entry lets the wallet answer "is this a token contract?" without heuristics. Important for policy: "only allow SEP-41 transfers up to N". |
| SEP-48 | Contract Interface Specification | Active (v1.1.0) [19] | A1 | Mitigates T3 (typed args with host types) | N5 | Richer-than-Wasm export descriptor (Soroban host types, user-defined types, events). Enables typed tool calls and human-readable preview of arbitrary contract invocations without hand-written ABIs. |
| SEP-53 | Sign and Verify Messages | Draft (v0.0.1) [20] | A3 | Mitigates T5 (proof-of-ownership off-chain); creates T9 if preview weak | N5 | Canonical ed25519 message signing (SHA-256 over prefix + message). Relevant to x402 receipts, off-chain attestation, social-platform proofs. Pairs with `signMessage` in SEP-43. |

[1–20] citations — see §6.

## 3. Relevance to an AI-agent wallet (rollup)

- **Actors served.** A1 (automation daemon), A2 (orchestrator), A3 (service-consumer), A4 (user-facing assistant), A5 (CI/CD), A7 (human operator). SEPs do not meaningfully serve A6 (read-only research) because read paths require no auth.
- **Threats mitigated.** T1 (SEP-5 scoped subaccounts, SEP-30 recovery); T3 (SEP-11 preview, SEP-41 typed interface, SEP-47/-48 typed contract calls); T5 (SEP-1 home-domain ID, SEP-10/-45 identity auth, SEP-29 memo check, SEP-46 contract identity); T9 (SEP-7 wallet-owned approval URI, SEP-43 typed signer API); T10 (every anchor/SEP flow carries network passphrase explicitly via `stellar.toml`).
- **Threats created.** T4 via SEP-41 `approve` allowances and SEP-30 recovery signers; T7 via SEP-1 fetches that depend on DNS/TLS; T9 via SEP-10 JWT handling if the wallet surfaces raw JWTs to the agent.
- **Non-negotiables touched.** N1 (self-custodial) by every SEP that leads to a signature; N2 (autonomous) by SEP-6/-10/-12/-24/-31/-38 which all depend on third-party anchor/service availability; N3 (no central server) by SEP-12 (do not cache PII server-side); N5 (deterministic output) is served by SEP-11, SEP-43, SEP-48.

Why these SEPs matter to this wallet, not just to Stellar in general: they define the **only standard surfaces** a self-custodial agent wallet has for interacting with anchors (fiat rails, quotes, cross-border), with identity-anchored counterparties (home domain, toml signing key), with contract accounts (SEP-45, SEP-41, SEP-48), and with human-owned approval channels (SEP-7, SEP-43). Skipping any of them means either reinventing the wheel non-interoperably or hand-coding per-counterparty integrations — both of which fail the interop goal of the RFP.

## 4. Agent-specific quirks

**SEP-10 under unattended context (A1, A3, A5).**
The challenge is a Stellar transaction with `sequenceNumber = 0`, a specific Manage-Data op, and signed by the server's `SIGNING_KEY` from `stellar.toml`. An unattended agent can complete the challenge because the structure is bounded and verifiable: the wallet validates sequence=0, first op is Manage-Data with key `<home_domain> auth`, the server signed it, optional `web_auth_domain` / `client_domain` ops have the right source accounts, no surprise operations are present [6]. This is the rare SEP flow where "the agent signs automatically" is safe, because the transaction is structurally inert — it has sequence 0, it cannot be submitted to the ledger, and the server verifies thresholds at `/token`. Still, **the wallet MUST enforce the structural validation locally** rather than trusting the server, because a malicious server could otherwise trick the agent into signing a real transaction encoded to look like a challenge.

**SEP-45 for `C...` accounts.**
Unlike SEP-10, SEP-45 signs **Soroban authorization entries** targeting a `web_auth_verify(...)` contract at the server's `WEB_AUTH_CONTRACT_ID`. The wallet must simulate the transaction and verify the ledger footprint contains only `ledger_key_nonce` read-writes on the expected set of addresses, plus optionally the archived web-auth contract instance [16]. If the footprint reveals any other write, the server is attempting a real side effect and the client MUST abort. A smart-account wallet (A1/A3 backed by a `C...` account) therefore depends on a working Soroban-RPC simulation path before any SEP-45 auth.

**SEP-24 purely-agent path (by design: none).**
SEP-24 requires the user to complete an anchor-hosted interactive form. A headless A1 cannot complete KYC unattended. The wallet's honest behaviour is: perform SEP-10/-45 auth programmatically, receive the interactive URL, emit a structured "requires human" response referencing A7, and stop. Do **not** attempt to scrape or automate the anchor's form — this is an anti-pattern that also breaks when the anchor changes UX.

**SEP-29 as a structural refusal rule.**
An agent can hallucinate a destination (T3) or be prompt-injected into one (T2). The wallet MUST fetch `config.memo_required` for every classic `G...` destination on `Payment`, `PathPayment*`, and `MergeAccount` ops, and refuse to sign without a memo [10]. This is a pre-signature check and cannot be delegated to the agent. For `M...` muxed destinations the check is unnecessary. For `C...` destinations the memo concept does not apply; parallel rules live in SEP-41 (wrong asset, wrong decimals).

**SEP-41 `approve` and allowance lifetimes.**
Soroban's SEP-41 `approve` takes `(spender, amount, live_until_ledger)` [14]. Unlike ERC-20 there is a ledger-expiry, which limits the blast radius of a stale approval — but an agent can still grant "effectively unlimited" by setting `i128::MAX` and a far-future ledger. The policy engine MUST treat `approve` as a first-class operation, capped distinctly from `transfer`, and SHOULD default `live_until_ledger` to a short window.

**SEP-41 is also the token substrate for agentic payments.** Both x402-on-Stellar and MPP (see `10-mpp.md`) denominate payments in SEP-41 contracts (SAC wrappers for USDC, EURC, XLM, and any other SEP-41-compliant token). The wallet's SEP-41 policy surface therefore extends to "an agentic-payment protocol just signed a `transfer` auth entry on my behalf" in addition to direct user-issued `transfer` calls — the same caps and identity-anchored allowlists apply to both.

**SEP-43 as the canonical external shape.**
`signTransaction`, `signAuthEntry`, `signMessage`, `getAddress`, `getNetwork` [15]. A wallet that exposes only an MCP surface will still be asked to integrate with Wallet Kit, Freighter, Lab. Providing a SEP-43-shaped adapter is cheap and unlocks the ecosystem tooling the operator already maintains (Flutter / iOS / PHP / KMP SDKs). Error codes (`-1` internal, `-2` upstream, `-3` bad request, `-4` user rejected) are a useful template for our own CLI/MCP error envelopes even if we do not ship a browser extension.

**SEP-46/-47/-48 and typed tool boundaries.**
T3 (hallucinated arguments) is the most constantly present failure mode of agent-driven contract calls. SEP-48 lets a contract ship its own interface descriptor including Soroban host types; SEP-47 lets a contract declare which SEPs it claims to implement (e.g. `sep=41`). The wallet can read these at call-time and present a typed, human-readable preview — and reject a call whose arguments do not satisfy the declared schema. This is the structural defence for T3 that the threat model demands (see `02-threat-model.md` §4 T3).

**Network scoping via SEP-1.**
`stellar.toml` has a `NETWORK_PASSPHRASE` field [1]. The wallet MUST cross-check the toml's passphrase against its current network context for every anchor/service interaction. Silent drift here is T10.

## 5. Status on the network

- **Mainnet.** SEP-1, -5, -6, -7, -9, -10, -11, -12, -24, -29, -31 in production with multiple anchors and wallets. SEP-38 and -41 are widely implemented in practice despite Draft status (Draft here means "text may still change", not "not deployed"). SEP-46 and -48 are Active in the spec and already used by `stellar-cli` and the Stellar contracts tooling.
- **Testnet.** Same coverage, plus SEP-45 reference implementations on testnet anchors during 2026.
- **Draft or pre-production.** SEP-30 (Draft since 2021), SEP-45 (Draft, iterating), SEP-47 (Draft), SEP-53 (Draft). These are safe to plan for but not safe to depend on without an escape hatch.
- **Roadmap / adjacent.** SEP-46/-47/-48 are the emerging tooling triad for Soroban contract discoverability. SEP-49 (upgradeable contracts), SEP-50 (NFTs), SEP-56 (tokenized vaults), SEP-57 (T-REX compliant tokens) are out of scope for the **core wallet** but may matter for contracts the wallet *calls* — handled in `04-smart-accounts.md` and `07-dex-amm.md`, not here.
- **Incidents.** No breaking SEP-level incidents in the last 12 months on the SEPs above; SEP-6 interactive components were deprecated in favour of SEP-24, which is a tidy-up not a break.

## 6. Primary documentation

All local citations are paths relative to [`stellar-protocol/`](https://github.com/stellar/stellar-protocol) at short SHA `44f5f5d`. Accessed 2026-04-18.

1. `ecosystem/sep-0001.md` — SEP-1 Stellar Info File, Active, v2.7.0, updated 2025-01-16.
2. `ecosystem/sep-0005.md` — SEP-5 Key Derivation Methods, Final, updated 2020-06-16.
3. `ecosystem/sep-0006.md` — SEP-6 Deposit and Withdrawal API, Active, v4.3.0, updated 2025-09-10.
4. `ecosystem/sep-0007.md` — SEP-7 URI Scheme, Active, v2.1.0, updated 2020-07-29.
5. `ecosystem/sep-0009.md` — SEP-9 Standard KYC Fields, Active, v1.17.0, updated 2024-04-22.
6. `ecosystem/sep-0010.md` — SEP-10 Stellar Web Authentication, Active, v3.4.1, updated 2024-03-20.
7. `ecosystem/sep-0011.md` — SEP-11 Txrep, Active, v1.1.0, updated 2021-10-09.
8. `ecosystem/sep-0012.md` — SEP-12 KYC API, Active, v1.15.0, updated 2024-07-23.
9. `ecosystem/sep-0024.md` — SEP-24 Hosted Deposit and Withdrawal, Active, v3.8.0, updated 2025-09-10.
10. `ecosystem/sep-0029.md` — SEP-29 Account Memo Requirements, Active, v0.5.0, updated 2020-05-04.
11. `ecosystem/sep-0030.md` — SEP-30 Account Recovery, Draft, v0.8.1, updated 2021-05-04.
12. `ecosystem/sep-0031.md` — SEP-31 Cross-Border Payments API, Active, v3.1.0, updated 2024-11-05.
13. `ecosystem/sep-0038.md` — SEP-38 Anchor RFQ API, Draft, v2.5.0, updated 2025-02-26.
14. `ecosystem/sep-0041.md` — SEP-41 Soroban Token Interface, Draft, v0.4.1, updated 2025-08-28.
15. `ecosystem/sep-0043.md` — SEP-43 Standard Web Wallet API Interface, Draft, v1.2.1, 2024-04-11.
16. `ecosystem/sep-0045.md` — SEP-45 Stellar Web Authentication for Contract Accounts, Draft, v0.1.1, updated 2025-12-16.
17. `ecosystem/sep-0046.md` — SEP-46 Contract Meta, Active, v1.0.0, updated 2025-04-16.
18. `ecosystem/sep-0047.md` — SEP-47 Contract Interface Discovery, Draft, v0.1.0, 2025-02-14.
19. `ecosystem/sep-0048.md` — SEP-48 Contract Interface Specification, Active, v1.1.0, updated 2025-04-16.
20. `ecosystem/sep-0053.md` — SEP-53 Sign and Verify Messages, Draft, v0.0.1, 2025-02-01.

Reference anchor implementation: [`anchor-platform/`](https://github.com/stellar/java-stellar-anchor-sdk) (SEP-6, -10, -12, -24, -31, -38).
Reference client implementation: [`typescript-wallet-sdk/`](https://github.com/stellar/typescript-wallet-sdk) and [`stellar-wallets-kit/`](https://github.com/Creit-Tech/Stellar-Wallets-Kit) (SEP-43).
Freighter SEP-43 adapter: [`freighter/`](https://github.com/stellar/freighter).

## 7. Known gaps or pain points

### 7.1 Out of scope for this wallet

- **SEP-31 (Cross-Border Payments API).** SEP-31 is the protocol between *two anchors*, not between a client and an anchor. Our wallet sits on the client side; supporting SEP-31 would only matter if the operator intends the wallet to operate as a sending or receiving anchor, which contradicts the self-custodial framing. Flagged as **OUT unless the operator confirms otherwise**.
- **SEP-24 as a fully automated agent flow.** SEP-24 is by construction interactive. Supporting it from A1 is impossible by design, not by omission. The wallet supports the SEP-10/-45 auth step programmatically and hands off the interactive URL to A7 via A4's approval channel.
- **SEP-49 / SEP-50 / SEP-56 / SEP-57 / SEP-52 (key sharing).** Contract-side standards (upgradeable contracts, NFTs, tokenized vaults, T-REX) belong in `04-smart-accounts.md` or `07-dex-amm.md`, not in the wallet's own SEP surface. SEP-52 (key sharing) is not agent-compatible without host-level threshold crypto that is out of scope.
- **SEP-02 (Federation), SEP-03 (deprecated).** Federation remains Active but serves human-facing address resolution (`alice*example.com`). Useful as a display nicety; not load-bearing. Listed here for completeness.

### 7.2 Open issues

- **SEP-30 is Draft and has limited production deployment.** Recovery is the primary N1-vs.-practicality trade-off for a self-custodial wallet. The spec is solid but the trusted-recovery-signer ecosystem is thin. Design SHOULD support SEP-30 but not REQUIRE a specific recovery provider.
- **SEP-41 `approve` lifetimes.** The ledger-expiry is a genuine improvement over ERC-20, but tooling to present "you are granting spender X the right to take up to Y tokens until ledger Z" in human terms is still immature. The wallet must render `live_until_ledger` as a wall-clock time estimate using ledger-close-time statistics.
- **SEP-45 dependency on Soroban-RPC simulation.** An unreachable RPC renders SEP-45 auth impossible. Our infra-ops capability file (`08-infra-ops.md`, if added) must cover multi-endpoint failover for this path.
- **SEP-43 is Draft.** Method signatures have mutated during 2024–2025 (notably `signAuthEntry` was added after initial publication). A wallet that advertises SEP-43 compliance must version-pin and document which draft it implements.
- **No SEP covers x402 on Stellar.** x402 is an HTTP-402-based payment negotiation orthogonal to the SEPs above. Authentication for x402 services will almost certainly layer on SEP-10/-45, but there is no SEP number for x402 itself as of 2026-04-18. Handled in the x402 research stream, not here.
- **Memo-type granularity.** SEP-29 only standardises "memo required" as a boolean. There is no standard for "memo must be a MEMO_ID of form N/M". Anchor-specific conventions still leak into client code.

## 8. Candidate requirements surfaced

- `REQ-sep-toml-fetch`: The wallet MUST fetch, validate (content-type, size ≤ 100KB, HTTPS, CORS), and cache `stellar.toml` per counterparty home domain before any SEP-dependent interaction, and MUST cross-check `NETWORK_PASSPHRASE` against the active network context. **MUST**.
- `REQ-sep-key-derivation`: The wallet MUST support SEP-5 BIP-39 mnemonic import and SLIP-10 derivation along `m/44'/148'/x'`, and MUST allow per-subaccount derivation for delegation (A2). **MUST**.
- `REQ-sep-10-client`: The wallet MUST implement SEP-10 as a client with full structural validation (sequence=0, Manage-Data ops, signing-key check, optional `client_domain` / `web_auth_domain`) **without** delegating validation to the server. **MUST**.
- `REQ-sep-45-client`: The wallet MUST implement SEP-45 as a client for `C...` accounts, including Soroban-RPC simulation and ledger-footprint validation before signing any auth entry. **SHOULD** (upgrade to MUST if smart-account-first is the chosen direction in `10-recommendation.md`).
- `REQ-sep-29-enforce`: The wallet MUST refuse to sign any `Payment`, `PathPaymentStrictSend`, `PathPaymentStrictReceive`, or `AccountMerge` operation targeting a `G...` account with `config.memo_required=1` when the transaction carries no memo. No override available to the agent; human override via explicit CLI flag only. **MUST**.
- `REQ-sep-7-inbound`: The wallet MUST accept SEP-7 `web+stellar:` URIs as signing requests and present them to A7 / A4 via the wallet-owned approval channel (never rendered by the agent). **SHOULD**.
- `REQ-sep-41-policy`: The wallet's policy engine MUST treat SEP-41 `approve` as a distinct operation class from `transfer`, with its own caps and a conservative default `live_until_ledger`. **MUST**.
- `REQ-sep-48-typed-preview`: The wallet SHOULD read SEP-46/-48 contract metadata when present and render typed, human-readable previews of contract invocations before signing. **SHOULD**.
- `REQ-sep-47-claim-check`: When the wallet's policy allowlists a contract category (e.g. "SEP-41 tokens only"), the wallet SHOULD verify the target contract's SEP-47 `sep` meta entry claims membership. **SHOULD**.
- `REQ-sep-43-adapter`: The wallet SHOULD expose a SEP-43-shaped API adapter for browser/web interop with `stellar-wallets-kit`, Freighter, Lab. **SHOULD**.
- `REQ-sep-53-message-sign`: The wallet SHOULD implement SEP-53 message signing with an explicit preview and per-origin policy, and MUST NOT reuse the signing path of real transactions. **SHOULD**.
- `REQ-sep-30-recovery-compat`: The wallet SHOULD interoperate with SEP-30 recovery signers (add/remove via account thresholds) but MUST NOT require a specific recovery service. **SHOULD**.
- `REQ-sep-24-handoff`: For SEP-24 flows the wallet MUST perform SEP-10/-45 auth programmatically and hand off the interactive URL to A7 via A4's approval channel; it MUST NOT attempt to scrape or automate the anchor's hosted form. **MUST**.
- `REQ-sep-6-optional`: SEP-6 programmatic deposit/withdraw SHOULD be supported for A3 where the counterparty does not require interactive KYC; graceful degradation to SEP-24 handoff otherwise. **SHOULD**.
- `REQ-sep-38-quotes`: Where SEP-38 quotes are available, the wallet SHOULD reference firm quotes by id and validate the quote's expiry and destination asset before signing. **SHOULD**.
- `REQ-sep-31-out`: SEP-31 integration is **OUT** of scope unless the operator confirms a specific use case in which the wallet plays an anchor role. **OUT**.

These flow into `analysis/03-requirements.md` via its candidates queue.

## 9. Cross-links

- Related capability files: `04-smart-accounts.md` (SEP-41 token interface and SEP-45 `C...` auth feed into this); `07-dex-amm.md` (Soroban AMMs trade SEP-41 tokens; uses SEP-47/-48); `03-soroban.md` (auth entries are the substrate for SEP-45 and SEP-41 `approve`); `08-infra-ops.md` if present (multi-endpoint RPC coverage needed for SEP-45).
- External candidates that interact with these SEPs: `research/external/stellar-ecosystem/freighter-walletkit.md` (SEP-43 reference); `research/external/stellar-ecosystem/stellar-mcp-server.md` (current agent surface for SEP-10); `research/external/stellar-ecosystem/meridian-pay.md` (uses SEP-10, SEP-41, SEP-45-adjacent smart accounts).
- Decision files that will cite this entry: `analysis/00-context.md` §5.3; `analysis/03-requirements.md` (all `REQ-sep-*` tags); `analysis/06-option-a-cli-first.md` and `analysis/07-option-b-agent-native.md` (SEP surface per option).
