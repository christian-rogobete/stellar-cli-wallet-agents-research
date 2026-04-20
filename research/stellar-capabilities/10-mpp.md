# MPP (Machine Payments Protocol) on Stellar

**Status:** in progress
**Last updated:** 2026-04-20
**ID:** 10-mpp

---

## 1. What it is

MPP is an open protocol for machine-to-machine payments over HTTP, co-developed by Tempo and Stripe. It extends the `402 Payment Required` status code into a formal three-primitive system: a **Challenge** (server signals payment requirement via `WWW-Authenticate: Payment`), a **Credential** (client submits payment proof via `Authorization: Payment`), and a **Receipt** (server confirms via `Payment-Receipt`). The specs repository is at [`tempoxyz/mpp-specs`](https://github.com/tempoxyz/mpp-specs); the core HTTP-Payment-Auth document is drafted under file name [`draft-httpauth-payment-00.md`](https://github.com/tempoxyz/mpp-specs/blob/main/specs/core/draft-httpauth-payment-00.md), submitted to the IETF Datatracker as an **individual submission** under the name [`draft-ryan-httpauth-payment`](https://datatracker.ietf.org/doc/draft-ryan-httpauth-payment/) (currently at version `-01`, not WG-track). The protocol is network-agnostic at its core and ships first-class method specs for Stellar, Lightning, Solana, EVM chains, Stripe, and card payments.

Stellar is one of the supported methods. The Stellar charge spec is [`draft-stellar-charge-00`](https://github.com/tempoxyz/mpp-specs/blob/main/specs/methods/stellar/draft-stellar-charge-00.md); the SDK is published as [`@stellar/mpp`](https://www.npmjs.com/package/@stellar/mpp) (MIT, v0.5.0 published 2026-04-15) under [`stellar/stellar-mpp-sdk`](https://github.com/stellar/stellar-mpp-sdk).

MPP is the second HTTP-402-based agentic-payment standard the wallet must interoperate with; the first is [x402](https://developers.stellar.org/docs/build/agentic-payments/x402) (Coinbase-origin, chain-agnostic, covered in `research/external/` and the x402 Stellar docs). MPP and x402 are siblings rather than competitors: they use different HTTP headers and bodies but can coexist on the same 402 response. Cloudflare's integration docs describe the overlap as "MPP is backwards-compatible with x402. The core x402 `exact` payment flows map directly onto MPP's `charge` intent."

## 2. Components

URLs accessed 2026-04-20.

### 2.1 Charge mode

Per-request on-chain Soroban SAC `transfer` invocation. Settles in roughly 5 seconds with deterministic Stellar finality. The primary mode for one-shot paid API requests.

**HTTP exchange** (four client-visible steps; the underlying Stellar charge draft's sequence diagram breaks server-side settlement into further sub-steps; see [`draft-stellar-charge-00`](https://github.com/tempoxyz/mpp-specs/blob/main/specs/methods/stellar/draft-stellar-charge-00.md)):

1. Client sends `GET /resource` without payment header.
2. Server responds `402 Payment Required` with:

   ```
   WWW-Authenticate: Payment id="<uuid>", realm="<host>", method="stellar",
       intent="charge", request="<base64url-JCS-JSON>", expires="<RFC3339>"
   ```

   Decoded `request` JSON carries `amount` (string, base units / stroops), `currency` (C-prefixed SAC contract address), `recipient` (Stellar account or contract), `methodDetails.network` (CAIP-2 `stellar:pubnet` or `stellar:testnet`), optional `methodDetails.feePayer` boolean.

3. Client retries with `Authorization: Payment <base64url-JSON>`. Credential JSON: `{ challenge: {...}, payload: { type: "transaction"|"hash", transaction?: "XDR-base64", hash?: "txhash" }, source?: "did:pkh:..." }`.
4. Server verifies, settles on-chain, returns `200 OK`; the core spec specifies the server **SHOULD** (not MUST) include `Payment-Receipt: <base64url-JSON>`: `{ status: "success", method: "stellar", reference: "<txhash>", timestamp: "<RFC3339>" }`. A wallet cannot rely on the receipt being guaranteed; settlement confirmation should fall back to on-chain lookup by the reference hash.

**Three client signing paths:**

- **Pull-sponsored** (`feePayer: true`): Client signs only the `SorobanCredentialsAddress` authorisation entry for `transfer(from, to, amount)` on the SAC contract. Transaction `source` is the all-zeros null account (`GAAAAAAAAAAAAAAA...AWHF`); server rebuilds the transaction with its own source and optional `FeeBumpTransaction` wrapper. Client does not need XLM.
- **Pull-unsponsored** (`feePayer: false`): Client builds and signs the complete classic transaction envelope (source, sequence, fee, `timeBounds.maxTime ≤ challenge.expires`). Server submits unmodified.
- **Push**: Client builds, signs, broadcasts independently; sends resulting transaction hash as the credential payload. Server polls `getTransaction` to confirm.

**Server verification checklist** (from the draft):

- Challenge ID matches an outstanding, unexpired challenge.
- Exactly one `invokeHostFunction` operation.
- The invoked function is `transfer(from, to, amount)`; recipient and amount match challenge parameters.
- Network passphrase matches.
- RPC simulation succeeds with only the expected balance changes; no unexpected third-party balance modifications.

**Errors.** `verification-failed` returns a fresh 402 with a new challenge. `settlement-failed` indicates the credential was valid but the on-chain transaction failed (insufficient funds, etc.). HTTP 5xx indicates Stellar RPC was unreachable during required simulation.

**Idempotency / replay.** Pull mode relies on auth-entry ledger expiration and sequence numbers; settled challenge IDs are rejected. Push mode maintains a consumed-hash set. The SDK persistence key format is `stellar:{intent}:{type}:{id}`.

### 2.2 Channel (session) mode

High-frequency mode using a **Soroban one-way payment-channel contract**. Client deposits once; signs cumulative off-chain commitments per request; server settles by closing the channel at any time. Intended for LLM token billing, streaming data feeds, and other workloads where per-request on-chain settlement would be prohibitively slow or costly.

**No formal spec draft exists.** The SDK at [`stellar/stellar-mpp-sdk`](https://github.com/stellar/stellar-mpp-sdk) is the reference; no `draft-stellar-channel-*` document has been submitted to `mpp-specs` as of 2026-04-20. Channel details documented here are drawn from SDK source and the Stellar developer docs.

**Open / deposit.** Client deploys or references an existing one-way-channel Soroban contract. Signs a channel-open transaction XDR (credential payload `action: "open"`). The contract escrows SEP-41 tokens on behalf of the funder.

**Off-chain commitment signing.** Per request (or per streaming chunk), the client calls `prepare_commitment(cumulativeAmount)` on the Soroban contract via RPC simulation. The contract returns raw `bytes` encoding the cumulative commitment. The client signs these bytes directly with ed25519 — using a dedicated commitment key, separate from the account's keypair — producing a 64-byte signature encoded as 128 hex characters (see [`Channel.ts` (client)](https://github.com/stellar/stellar-mpp-sdk/blob/main/sdk/src/channel/client/Channel.ts)). Credential payload per sub-request: `{ action: "voucher", amount: "<cumulative-stroops>", signature: "<128-hex>" }`.

**Anti-reset.** SDK client persists the last-signed cumulative amount locally and takes `max(local, server-reported)` before signing. This defeats a malicious server that resets the baseline downward to extract over-payment. SDK-level, not spec-level — and the SDK's default store is **in-memory** (introduced as an optional client-side store in `0.4.0`), so the guarantee does not survive a process restart unless the implementer wires a persistent store. Wallets carrying long-running channels across restarts must provide their own durable store.

**Settlement / close.** Server invokes the contract's `close(commitmentAmount: i128, signature: bytes)` with the highest valid cumulative commitment and its signature (see [`Channel.ts` (server)](https://github.com/stellar/stellar-mpp-sdk/blob/main/sdk/src/channel/server/Channel.ts)). Contract transfers the committed amount to the recipient and auto-refunds remaining balance to the funder.

**Close-override.** The `Methods.ts` schema has `action: "close"` as a client-initiated credential type. Whether a funder can force close with a time-lock dispute period is not documented in the accessible materials — see §7.

### 2.3 MCP transport binding

MPP defines a native MCP / JSON-RPC binding. MPP-enabled service hosts return JSON-RPC error code `-32042` to signal payment is required. Agents include proof in `_meta.org.paymentauth/credential` on the next tool call; receipts come back in `_meta.org.paymentauth/receipt`. The payment negotiation happens in-band within the MCP tool-call protocol; no external HTTP round-trip is needed. See [`mpp.dev/protocol/transports/mcp`](https://mpp.dev/protocol/transports/mcp).

There is **no standalone MCP server for MPP**. The MCP transport is a binding that MPP-enabled service hosts implement. For the wallet, this means MPP-speaking agents encounter payment challenges inside their existing tool calls and need credential-construction capability in that path.

### 2.4 Discovery

Two-tier. **Out-of-band (primary):** services publish an OpenAPI 3.1 document at `/openapi.json` with `x-payment-info` (per-operation: amount, currency, intent, method) and `x-service-info` (service-level metadata, including `llms.txt` pointer). **In-band (authoritative):** the 402 `WWW-Authenticate: Payment` challenge is the binding source of truth for a given request; discovery documents are "informational hints" per [`mpp.dev/protocol/discovery`](https://mpp.dev/protocol/discovery). The `Accept-Payment` request header lets clients declare supported method / intent combinations so servers can filter which challenges to return.

### 2.5 Coexistence with x402

A single 402 response can carry both MPP and x402 header sets simultaneously. x402 clients ignore `WWW-Authenticate`; MPP clients ignore the `X-` headers.

| Aspect | x402 | MPP |
|---|---|---|
| Challenge header | `X-PAYMENT-REQUIRED` | `WWW-Authenticate: Payment` |
| Credential header | `X-PAYMENT` | `Authorization: Payment` |
| Receipt header | `X-PAYMENT-RESPONSE` | `Payment-Receipt` |
| Error format | Custom JSON | RFC 9457 Problem Details |
| Session support | V2 (Dec 2025) | Yes (channel intent) |
| Multi-method | No | Yes (Stripe, Lightning, Solana, card, ...) |
| IETF track | No | Yes (`draft-httpauth-payment-00`) |

Per [`mpp.dev/guides/upgrade-x402`](https://mpp.dev/guides/upgrade-x402) and [`developers.cloudflare.com/agents/agentic-payments/mpp/`](https://developers.cloudflare.com/agents/agentic-payments/mpp/), MPP frames x402 as a predecessor offering "multi-method payments, sessions, and IETF standardization" as the upgrade value. The Stellar developer docs treat both as parallel-track standards the ecosystem supports concurrently.

### 2.6 Stellar signing primitives involved

- **SAC tokens first-class.** The `@stellar/mpp` SDK exports named constants for USDC and XLM SAC addresses; any SEP-41-compliant token is supported. The `currency` field carries the C-prefixed SAC contract address.
- **Charge auth entry.** `SorobanCredentialsAddress` variant of `SorobanAuthorizationEntry`, signing an `InvokeContractAuth` for `transfer(from: Address, to: Address, amount: i128)` on the SAC. Spec requires exactly one `invokeHostFunction` and exactly the `transfer` function.
- **Channel commitment signing.** Not a standard auth entry. Client signs raw bytes (obtained from `prepare_commitment` via simulation) with a dedicated ed25519 key. This is a distinct signing operation from anything in `03-soroban.md`'s `SorobanAuthorizationEntry` inventory.
- **Classic envelope.** Used only in pull-unsponsored and push paths.

## 3. Relevance to an AI-agent wallet

- **Actors served** (from `analysis/01-actor-model.md`): A1 (unattended automation daemon), A2 (multi-agent orchestrator), A3 (service-consumer, x402 / SEP-10), A5 (CI/CD deploy agent paying for deployment-adjacent APIs).
- **Non-negotiables touched** (from `analysis/00-context.md` §4): N1 (self-custodial signing), N2 (autonomous operation — no project-operated backend between agent and Stellar), N4 (permissive licence — MIT SDK, CC0 spec, no AGPL concern), N5 (deterministic, scriptable output), N6 (testnet / mainnet parity via CAIP-2).
- **Threats mitigated or created** (from `analysis/02-threat-model.md`): T2 (prompt-injected transactions — MPP has no human-in-the-loop gate in the spec; this is a created threat the wallet must defend against externally), T3 (hallucinated arguments — typed Zod schemas in the SDK mitigate), T4 (delegated-authority abuse — channel commitment keys create a new delegation surface with undefined management), T5 (counterparty impersonation — MPP's `source: did:pkh` and `realm` fields need identity-anchored policy), T9 (approval-channel manipulation — MPP has no approval channel; for A4 the wallet must impose one externally).

MPP is the second Stellar-supported HTTP-402-based machine-payment protocol. A compliant agent wallet must implement MPP alongside x402 (see `research/stellar-capabilities/05-seps.md` for the SEP-41 dependency that both protocols rely on). Charge mode fits naturally into the Soroban auth-entry surface described in `03-soroban.md` §`REQ-sor-auth-entries`. Channel mode is a genuinely new capability: a dedicated commitment key, off-chain cumulative-commitment signing, and a payment-channel Soroban contract the wallet must either interact with or avoid safely.

## 4. Agent-specific quirks

- **Two different signing keys for one wallet session.** Charge mode signs auth entries with the account's keypair (or via smart-account `__check_auth`). Channel mode signs commitment bytes with a *separate* ed25519 key. A wallet managing multiple concurrent channels has to manage multiple commitment keys. The SDK assumes one commitment key per channel instance; key derivation, rotation, and revocation are not specified.
- **No human-in-the-loop hook.** The spec has no concept of "this payment requires user approval." An agent under prompt injection can trigger legitimate channels and legitimate charges with valid credentials. The wallet has to impose policy (per-tx caps, per-period caps, counterparty allowlists) externally; MPP itself provides no defence.
- **RPC liveness as a hard dependency.** Both modes require live Soroban RPC (charge for server re-simulation, channel for `prepare_commitment` byte derivation). No fallback; the spec mandates HTTP 5xx on RPC failure. A wallet operating in a degraded RPC environment cannot participate in either mode.
- **Receipt-but-no-service (403-after-200) has no refund path.** If payment settles but the server then returns 403 (access denied, policy mismatch, server re-evaluation), there is no defined chargeback mechanism. The wallet sees a settled tx with no delivered resource; the agent has spent budget with nothing to show.
- **Commitment-baseline attack is SDK-mitigated, not spec-mitigated.** A compliant but naive wallet implementation that does not persist `max(local, server-reported)` will over-sign under a malicious server. Implementers must replicate the SDK's anti-reset discipline explicitly.
- **Discovery is informational, not authoritative.** An agent that pre-plans budget from the OpenAPI `x-payment-info` and trusts it without re-reading the 402 challenge will be wrong when services change pricing mid-session. The wallet's policy evaluation must key off the challenge body, not the discovery document.
- **Single-signer SDK assumption.** The `resolveKeypair` and `envelopeSigner` utilities assume one keypair. Multisig accounts, passkey-backed smart accounts, and policy-signer patterns (the exact patterns an agent wallet's smart-account delegation wants) are **not first-class** in the current SDK. Custom credential construction outside the SDK is required.
- **Pull-sponsored path requires a null source account.** The `GAAAAAAAAAAAAAAA...AWHF` convention is non-obvious for a Stellar-native developer who would typically set source to the transaction-originator. The wallet must special-case this in its tx-construction path.

## 5. Status on the network

- **Mainnet:** charge mode usable via the `@stellar/mpp` SDK (v0.5.0). Channel mode implemented in SDK, testable on the `mpp.stellar.buzz` demo (charging 0.01 USDC/request) but no formal channel spec in `mpp-specs`. SAC-denominated payments in production (USDC, XLM wrapped).
- **Testnet:** same implementation.
- **Roadmap.** The `mpp-specs` repo is actively iterating; the Stellar method spec file carries `-00` suffix (the core spec's IETF Datatracker submission `draft-ryan-httpauth-payment` is already at `-01`). SDK has published five versions (`0.2.0`, `0.2.1`, `0.3.0`, `0.4.0`, `0.5.0`) across early 2026; not API-stable. A formal channel draft (`draft-stellar-channel-00` or equivalent) is an expected next step.
- **Known incidents:** none reported. No CVEs as of 2026-04-20. HN discussion ([`news.ycombinator.com/item?id=47426936`](https://news.ycombinator.com/item?id=47426936)) raised prompt-injection concerns as a class risk; not a defect but a named gap.

## 6. Primary documentation

All URLs accessed 2026-04-20.

- Landing and protocol docs: [`mpp.dev`](https://mpp.dev), [`mpp.dev/overview`](https://mpp.dev/overview), [`mpp.dev/protocol/http-402`](https://mpp.dev/protocol/http-402), [`mpp.dev/protocol/transports/mcp`](https://mpp.dev/protocol/transports/mcp), [`mpp.dev/protocol/discovery`](https://mpp.dev/protocol/discovery), [`mpp.dev/guides/upgrade-x402`](https://mpp.dev/guides/upgrade-x402).
- Stellar method docs on MPP: [`mpp.dev/payment-methods/stellar`](https://mpp.dev/payment-methods/stellar), [`mpp.dev/payment-methods/stellar/charge`](https://mpp.dev/payment-methods/stellar/charge), [`mpp.dev/payment-methods/stellar/session`](https://mpp.dev/payment-methods/stellar/session).
- Stellar developer docs: [`developers.stellar.org/docs/build/agentic-payments`](https://developers.stellar.org/docs/build/agentic-payments), [`developers.stellar.org/docs/build/agentic-payments/mpp`](https://developers.stellar.org/docs/build/agentic-payments/mpp).
- IETF / specs repo: [`tempoxyz/mpp-specs`](https://github.com/tempoxyz/mpp-specs); core draft [`draft-httpauth-payment-00`](https://github.com/tempoxyz/mpp-specs/blob/main/specs/core/draft-httpauth-payment-00.md); Stellar charge draft [`draft-stellar-charge-00`](https://github.com/tempoxyz/mpp-specs/blob/main/specs/methods/stellar/draft-stellar-charge-00.md); charge intent draft [`draft-payment-intent-charge-00`](https://github.com/tempoxyz/mpp-specs/blob/main/specs/intents/draft-payment-intent-charge-00.md); [`paymentauth.org`](https://paymentauth.org) (IETF draft rendering).
- Reference implementation: [`stellar/stellar-mpp-sdk`](https://github.com/stellar/stellar-mpp-sdk); charge Zod schema at [`sdk/src/charge/Methods.ts`](https://github.com/stellar/stellar-mpp-sdk/blob/main/sdk/src/charge/Methods.ts); channel client signing at [`sdk/src/channel/client/Channel.ts`](https://github.com/stellar/stellar-mpp-sdk/blob/main/sdk/src/channel/client/Channel.ts); channel server settlement at [`sdk/src/channel/server/Channel.ts`](https://github.com/stellar/stellar-mpp-sdk/blob/main/sdk/src/channel/server/Channel.ts). npm package at [`@stellar/mpp`](https://www.npmjs.com/package/@stellar/mpp) (metadata via `npm info`; web page returned 403 on 2026-04-20).
- Ecosystem context: [`developers.cloudflare.com/agents/agentic-payments/mpp/`](https://developers.cloudflare.com/agents/agentic-payments/mpp/); [`stellar.org/blog/foundation-news/x402-on-stellar`](https://stellar.org/blog/foundation-news/x402-on-stellar) (relates x402 and MPP positioning).
- Community discussion: [`news.ycombinator.com/item?id=47426936`](https://news.ycombinator.com/item?id=47426936) (prompt-injection concerns); [`forrester.com/blogs/why-stripes-machine-payments-protocol-signals-a-turning-point-for-micropayments/`](https://www.forrester.com/blogs/why-stripes-machine-payments-protocol-signals-a-turning-point-for-micropayments/) (industry analysis).

Licences: spec text CC0 1.0 Universal; tooling in `mpp-specs` Apache-2.0 OR MIT; `@stellar/mpp` npm and `stellar/stellar-mpp-sdk` repo MIT (confirmed via `npm info`; `LICENSE` file at the repo root not directly fetched).

## 7. Known gaps or pain points

Ordered by impact on a wallet implementer.

1. **No formal channel spec.** SDK implements channel mode; `mpp-specs` has no named draft. The protocol's security properties, dispute resolution, funder unilateral-close mechanism, and time-lock semantics are not normatively specified. Building against the SDK today means building against a moving target.
2. **No dispute / funder-reclaim time-lock documented.** The channel contract auto-refunds remaining balance to the funder on `close`, but there is no documented mechanism for a funder to reclaim deposits if the server becomes unresponsive or refuses to close. A self-custodial wallet cannot safely deposit into an unknown channel without this assurance; implementers need either an upstream spec change or a local policy that avoids long-lived channels with unaudited servers.
3. **Commitment-key management undefined.** A wallet running multiple concurrent channels across multiple agents has to manage multiple commitment keys. No guidance from spec or SDK on derivation, rotation, revocation, or scope. Likely maps to something like SEP-5 subaccount derivation, but that is a wallet-side design decision.
4. **No human-in-the-loop authorisation boundary in the spec.** Prompt injection to legitimate payment flows with valid credentials is a named community concern with no MPP-side mitigation. The wallet has to impose policy externally; the spec does not help.
5. **No refund path for 403-after-payment.** Payment settles, server denies delivery. Mapping this to a meaningful user-facing outcome is a wallet-side problem.
6. **Atomicity of verify + deliver undefined.** The core spec notes atomic verification and resource delivery are required but leaves the implementation to servers. A wallet receiving HTTP 5xx after credential submission cannot know whether the payment settled. The push-mode consumed-hash set and pull-mode settled-challenge-ID set help, but the wallet has to reconcile from the ledger, not from the server.
7. **SDK version instability.** Five SDK versions in rapid succession in 2026-04; package rename from `stellar-mpp-sdk` to `@stellar/mpp`. Not API-stable. Pinning a version and tracking upstream is the only safe path.
8. **No multi-signer / smart-account path in the SDK, though the composition exists architecturally.** When the `from` address in an MPP charge credential is a C-account (OZ `stellar-accounts` or custom), the `SorobanCredentialsAddress` auth entry invokes the account's `__check_auth`, which routes through its context-rule / signer / policy tree naturally — the same way any Soroban call does. The wiring gap is in `@stellar/mpp`: the SDK assumes single-keypair signing and does not construct the `AuthPayload` with `context_rule_ids` or route signatures to verifier contracts. A wallet integrating MPP charge under policy (e.g. an AI-agent context rule with spending-limit policy, matching the pattern in the [OZ `stellar-accounts` README §"Use Cases / AI Agents"](https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/README.md)) has to construct the credential outside the SDK. This is a wiring issue, not a protocol or architecture blocker.
9. **Pull-sponsored path's null source account convention.** `GAAAAAAAAAAAAAAA...AWHF` is non-obvious and easy to get wrong. A wallet must special-case this in its transaction-construction path or mislabel the signature intent to the user.
10. **RPC liveness dependency without fallback.** Both modes require live Soroban RPC. In a degraded RPC environment the wallet cannot construct valid credentials at all.
11. **IETF draft is version-00.** IANA registries (HTTP Payment Methods, HTTP Payment Intents) referenced in the spec do not yet exist as formal registries. Any IETF process change could affect header names and semantics before stabilisation.
12. **One-way-channel Soroban contract source location not discoverable from the SDK.** The SDK references "one-way-channel contract deployed on Stellar" without pointing at its source repository. Audit status and upgrade governance of the channel contract are unknown; a wallet-side integration has to trust the deployed bytecode without direct source review. This is an open vendor-clarification item.

## 8. Candidate requirements surfaced

- `REQ-svc-mpp-charge`: the wallet supports MPP charge mode (all three paths: pull-sponsored, pull-unsponsored, push). Signs `SorobanCredentialsAddress` auth entries for the `transfer(from, to, amount)` host function on SAC contracts. Validates challenge parameters (amount, recipient, currency, `expires`, network) before signing. **MUST.** Source: this file §2.1, `02-threat-model.md#T3`.
- `REQ-svc-mpp-channel`: the wallet supports MPP channel mode: channel open, cumulative off-chain commitment signing with a dedicated ed25519 commitment key, anti-reset discipline (persist `max(local, server-reported)` cumulative), client-initiated close. **SHOULD.** (Not MUST because the channel spec is not yet formalised.) Source: this file §2.2, §7 item 1.
- `REQ-svc-mpp-commitment-keys`: the wallet manages commitment keys per channel with deterministic derivation (SEP-5-style, with a channel-specific path prefix) and scoped revocation. Keys are not reusable across channels. **SHOULD.** Source: this file §4, §7 item 3.
- `REQ-svc-mpp-null-source-account`: the wallet correctly constructs and signs pull-sponsored transactions using the `GAAAAAAAAAAAAAAA...AWHF` null source account convention; surfaces this to the user or agent as a sponsored-path indicator, not as an unknown-source warning. **MUST.** Source: this file §2.1 pull-sponsored, §7 item 9.
- `REQ-svc-mpp-challenge-validation`: the wallet re-decodes and validates the 402 challenge body against the OpenAPI discovery document (if available) before signing; mismatches trigger a policy decision, not silent acceptance. **SHOULD.** Source: this file §2.4, §4.
- `REQ-svc-mpp-policy-gating`: MPP charge and channel operations are evaluated by the local policy engine (per-tx caps, per-period caps, counterparty allowlists, rate limits) before credential construction; the spec has no human-in-the-loop gate, so all authorisation discipline lives in the wallet. **MUST.** Source: this file §3, §7 item 4, `02-threat-model.md#T2`.
- `REQ-svc-mpp-receipt-audit`: every successful MPP credential-submission produces an audit record in the hash-chained audit log, correlating challenge ID, on-chain transaction hash (or channel commitment sequence), server receipt timestamp, and resource URL. **SHOULD.** Source: this file §2.1, `05-seps.md#8` (audit-log linkage).
- `REQ-svc-mpp-x402-coexistence`: when a server publishes both MPP and x402 challenges on one 402 response, the wallet chooses one protocol per request rather than dispatching both. The selection rule is an implementation choice; one reasonable default is MPP given its IETF-track submission and session-mode support, with x402 as fallback. **SHOULD.** Source: this file §2.5.
- `REQ-svc-mpp-version-pinning`: the wallet pins a specific `@stellar/mpp` SDK version and a specific spec draft version; version bumps are a wallet release event, not a silent dependency refresh. **SHOULD.** Source: this file §5, §7 item 7.
- `REQ-svc-mpp-mcp-transport`: if the wallet hosts an MCP server, it handles the MPP JSON-RPC `-32042` error code and the `_meta.org.paymentauth/credential` / `_meta.org.paymentauth/receipt` metadata fields as first-class MCP protocol events. **SHOULD.** Source: this file §2.3.
- `REQ-svc-mpp-channel-dispute`: if and when a formal channel dispute / funder-reclaim path is specified, the wallet exposes it as a first-class operation (channel-force-close by funder, with time-lock). **NICE.** Blocked on spec availability. Source: this file §7 item 2.
- `REQ-svc-mpp-smart-account-composition`: when an MPP charge credential's `from` address is a C-account, the wallet constructs the auth entry's `AuthPayload` with `context_rule_ids` selected according to the account's configured rules and routes signatures through the account's signer set (including `External` verifiers). This makes MPP charge operations governed by the same context-rule / signer / policy tree as any other Soroban call on that account. **SHOULD.** Source: this file §7 item 8, `04-smart-accounts.md` §2.

These flow into `analysis/03-requirements.md` via its candidates queue and into `03-requirements-addendum.md` for the stream-2 reconciliation.

## 9. Cross-links

- Related capability files: `03-soroban.md` (`SorobanAuthorizationEntry` shape used by charge mode), `04-smart-accounts.md` (smart-account composition with MPP is an open gap — current SDK assumes single-signer), `05-seps.md` (MPP uses SEP-41 / SAC tokens), `08-infra-ops.md` (channel settlement as a submission pattern), `07-dex-amm.md` (token-identity canonicalisation between classic issuer triple and SAC contract applies to MPP's `currency` field).
- External candidates (under `research/external/`): none directly. x402 coverage is in `research/external/_summary.md` non-wallet findings and `developers.stellar.org/docs/build/agentic-payments/x402`; MPP is the Stellar-native sibling.
- Analysis files that cite this entry: `analysis/03-requirements-addendum.md` §5.10 (`REQ-svc-mpp-*` cluster); `analysis/06-option-a-cli-first.md` §5 (A3 actor surface), §7 (SEP / Soroban coverage), §10 (cost estimate row); `analysis/07-option-b-agent-native.md` §5 (A3 actor surface) and §7 (chain-profile MPP paragraph); `analysis/01-actor-model.md` A3; `analysis/00-context.md` §5.3.
