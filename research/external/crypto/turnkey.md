# Turnkey

**Status:** complete
**Researcher:** research-agent (opus-4.7)
**Last updated:** 2026-04-18
**ID:** turnkey
**Category:** crypto-agent

---

## 1. What it is

Turnkey is a hosted key-management platform. Private keys are generated and held server-side inside AWS Nitro Enclaves running a purpose-built Linux unikernel called QuorumOS; users authenticate to Turnkey with passkeys, API keys, OAuth or email, and every signing request is evaluated by a policy engine inside the enclave before the enclave produces a signature [1][2][3]. The product is sold as either "embedded wallets" (one Turnkey sub-organization per end user) or "company wallets" (policy-governed workflows for teams), with explicit support for provisioning wallets to AI agents [4][5].

## 2. Agent interface

HTTP API plus TypeScript, Rust and Dart SDKs (all under `github.com/tkhq`) [6]. The minimum programmatic sequence for a signed Stellar payload: (a) client generates or loads a P-256 API key; (b) client stamps an HTTP request with that key; (c) Turnkey verifies the stamp against the organization's users, runs the activity through the policy engine inside an enclave, and (d) returns a raw signature. Stellar is not a first-class "Tier 4" ecosystem in Turnkey's terms — agents use the curve-level `ACTIVITY_TYPE_SIGN_RAW_PAYLOAD_V2` against an Ed25519 wallet account, and assemble the Stellar envelope client-side [2][7][8].

## 3. Custody model

Fully server-side custody of signing keys, with the end user holding only an authenticator, not a key share.

- **Where keys live.** HD seeds and private keys are generated and stored inside AWS Nitro Enclaves running QuorumOS; they are never exported to Turnkey staff, the end user, or the calling application [1][3]. Curves: secp256k1, Ed25519, P-256 [9]. Stellar addresses are not a first-class derivation type but any Ed25519 wallet account can sign arbitrary Stellar transaction hashes via raw-payload signing [2][7].
- **Who can sign.** Anyone whose authenticator satisfies a policy's `consensus` clause, subject to activity-level `condition` clauses, plus the root quorum which bypasses the policy engine entirely [10][11].
- **User device.** Holds only authenticators: a WebAuthn passkey, a long-lived P-256 API key, or a short-lived session API key (default 900 s, configurable, max 10 concurrent expiring keys per user) [12]. The authenticator signs *requests to Turnkey*; Turnkey signs *transactions to the chain*.
- **Agent host.** Same surface as a user — typically a long-lived or session API key scoped by a sub-organization's policies.
- **Turnkey infrastructure.** Holds all wallet seeds and private keys, runs the policy engine, and produces every chain signature.

"Non-custodial" in Turnkey's framing means "Turnkey cannot sign without an authenticator-stamped request that satisfies policy" [5]. It does **not** mean the user holds signing-key material; a Turnkey outage or coercion of Turnkey's operators blocks the user from signing, which is the relevant distinction for N1 below.

## 4. Policy and approval model

Policies are JSON objects with three fields — `effect` (`EFFECT_ALLOW` / `EFFECT_DENY`), `consensus` (who must approve), and `condition` (what the activity must look like) — evaluated by a domain-specific expression language inside the enclave [10][13]. Evaluation order: root quorum short-circuits; otherwise any `EFFECT_DENY` wins, else at least one `EFFECT_ALLOW` must match, else implicit deny [10].

Grammar highlights [13]:

- Operators: `&&`, `||`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `in`; indexing `x[i]`, slicing `x[0..2]`, field access `x.field`.
- Aggregations: `.all(item, pred)`, `.any(item, pred)`, `.contains(v)`, `.count()`, `.filter(item, pred)`.
- Namespaces: `activity` (`type`, `resource`, `action`, `params`), `approvers`, `credentials`, `wallet`, `wallet_account` (`.address`), `private_key`, and chain namespaces `eth.tx`, `eth.eip_712`, `eth.eip_7702_authorization`, `solana.tx`, `bitcoin.tx`, `tron.tx`, `tempo.tx`.
- No `stellar.tx` namespace; Stellar transactions reach the engine as `sign_raw_payload` activities exposing only `hash_function` and `encoding`, so destination/amount policy cannot be expressed declaratively for Stellar today [7][13].
- Integers are i128; the engine does not short-circuit and errors on invalid clauses [13].

Example rules in the wild [13][14]:

```
consensus: "approvers.any(user, user.id == '<USER_ID>')"
condition: "activity.resource == 'WALLET' && activity.action == 'CREATE'"
```
```
consensus: "approvers.filter(user, user.tags.contains('<TAG>')).count() >= 2"
```
```
condition: "eth.tx.to == '0x…' && eth.tx.value < 1000000000000000000"
```
```
condition: "solana.tx.transfers.all(t, t.to == '<ADDR>')"
```

No first-class time-window or per-period frequency operator is documented: there is no `activity.timestamp` and no rate primitive [13]. Time-bound authorization is instead expressed through expiring session API keys (per-user TTL, default 15 min, max 10 concurrent) [12]. Enforcement is vendor-side inside the enclave; a compromised Turnkey signing pipeline bypasses all policy. Policy changes are themselves activities and can be gated by root-quorum consensus [11].

## 5. Delegation model

Delegation is "issue another credential with a narrower policy," not key sharing. A parent organization creates a sub-organization per end user or per agent; policies and users are fully segregated [4]. A sub-agent receives either a long-lived API key or a short-lived session key; either can be revoked by a policy or user-delete activity. Revocation takes effect on the next request because every request re-enters the policy engine. Root-quorum members bypass the engine, so root membership is the real delegation unit of last resort [11].

## 6. Relevant protocol coverage

- **Curves:** secp256k1, Ed25519, P-256 [9].
- **Chains with Tier-4 transaction parsing/policy:** EVM, Solana (SVM), Bitcoin, Tron [7][13].
- **Stellar:** not listed among ecosystem packages [9]; Turnkey's whitepaper names Stellar only as an example Ed25519 chain supported at the curve level [2]. Practical path is `SIGN_RAW_PAYLOAD_V2` over Ed25519 + client-side Stellar XDR assembly; SEPs, Soroban auth entries, and smart-account integration are entirely out of band from Turnkey's policy engine.
- **Authenticators:** WebAuthn passkeys, P-256 API keys, OAuth, email, SMS (enterprise); sessions via P-256 ephemeral keys stored in IndexedDB (web) or secure storage (mobile) [12][15].

## 7. Licence and adoption signals

- **Service licence:** proprietary SaaS [5].
- **Open-source components:** TypeScript SDK (Apache-2.0), Rust SDK (Apache-2.0), Dart SDK, QuorumOS (AGPL-3.0), `core-enclaves` PCR index (AGPL-3.0) [6][16].
- **Self-hostable:** no. QuorumOS is open-source and independently buildable, but the Turnkey service — policy engine binaries, key-provisioning ceremony, PKI, dashboard — is not distributed [6][16][17].
- **Source available:** partial.
- **Last meaningful commit:** QuorumOS latest release `qos_client-v0.6.1` dated 2026-04-09 [16].
- **GitHub stars:** `tkhq/sdk` 95, `tkhq/qos` 102, `tkhq/rust-sdk` 30 [6][16].
- **Production users cited:** Moonshot, Spectral Labs [18][19].
- **Commercial backing:** venture-funded company.
- **Spec adherence:** no Open Wallet Standard support documented; no SEP coverage.
- **Pricing:** Free (100 wallets, 25 signatures/mo); Pay-as-you-go ($0.10/signature, up to 1 000 wallets); Pro ($99/mo, $0.05/signature, 2 000 wallets); Enterprise (down to $0.0015/signature, unlimited wallets, SMS auth, gas sponsorship) [20].

## 8. Non-negotiable check

Against `analysis/00-context.md` §4:

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | **no** | Signing keys live permanently inside Turnkey-operated Nitro Enclaves; the user holds only an authenticator that requests signatures. Turnkey-side compromise or outage blocks signing, and Turnkey is the sole producer of on-chain signatures [1][3][5]. |
| N2 | Autonomous (no project-operated backend) | **no** | Every signature requires a live call to `api.turnkey.com` [1][3]. |
| N3 | No central server for keys/policy/history | **no** | Keys, policies, activity history and authenticator metadata all live in Turnkey's infrastructure [1][10][11]. |
| N4 | Open source, permissive licence | **partial** | SDKs are Apache-2.0; QuorumOS and enclave manifests are AGPL-3.0; the service binaries are closed [6][16]. |
| N5 | JSON-default output | yes | HTTP+JSON API; activities and policies are JSON [1][10]. |
| N6 | Testnet/mainnet parity | n/a | Turnkey is chain-agnostic at the curve level; network is a client-side concern [2]. |

N1–N3 fail by construction. Turnkey is therefore a **design reference only**, not a deployable custody path for this project.

## 9. What to adopt

- **Activity × resource × action taxonomy.** A single `{effect, consensus, condition}` triple typed over `(activity.type, activity.resource, activity.action)` is a compact policy shape we can reuse locally to gate Stellar operations (`op == PAYMENT && asset == NATIVE && amount <= X`). Cite Turnkey §4, examples [13][14].
- **Root-quorum escape hatch with explicit semantics.** An "root bypass" path that skips the policy engine, documented as the only way to recover from a policy lockout, is a pattern our local wallet needs for human break-glass (A7) [11].
- **Expiring session credentials instead of long-lived keys.** 15-minute P-256 session keys stored in non-exportable browser/OS key stores, capped at 10 concurrent, are a useful model for agent-session scoping on top of a locally held root key [12][15].
- **Reproducible, attestable execution surface.** QuorumOS + StageX reproducible builds demonstrate that remote attestations are only meaningful with reproducible EIFs; if we ever offer an optional hardened-enclave adjunct, this is the bar [17].
- **Raw-payload signing as the curve-level fallback.** Their `SIGN_RAW_PAYLOAD_V2` cleanly separates curve ops from chain semantics; we should keep an equivalent low-level signing primitive for chains we have not yet parsed [13].

## 10. What to avoid

- **Vendor-side policy evaluation as the only policy locus.** Against T2/T4: a prompt-injected agent can do anything the session API key's policy allows, and the user has no local second opinion before the enclave signs [10].
- **"Self-custodial" framed as "vendor cannot sign without approval."** Direct N1 failure; end user never holds key material, recovery depends on Turnkey's continued operation [5].
- **Hard dependency on a live SaaS endpoint for every signature.** Breaks N2/N3 and introduces a strong liveness assumption incompatible with A1/A5 unattended profiles.
- **No time-window, per-period or rate-limit operators in the policy grammar.** T6 (runaway loops) can be bounded only indirectly via session TTL [12][13].
- **No Stellar-aware transaction parsing in the policy engine.** Destination, asset, memo, operation type and Soroban auth-entry structure cannot be asserted declaratively; only raw-payload metadata is visible to rules [7][13].
- **AGPL on the only self-buildable component (QuorumOS).** Adoption-unfriendly for a permissively licensed reference wallet [16].

## 11. Open questions

- Does Turnkey expose Ed25519 raw-signing metadata (e.g. Stellar transaction hash prefix) in any form that would let `sign_raw_payload` policies distinguish networks? Docs surveyed do not show this.
- Is there a documented, supported path for co-signed Stellar multisig where Turnkey is one of N signers? Curve support suggests yes, but no official Stellar ecosystem page exists.
- Can QuorumOS + `core-enclaves` be operated end-to-end without the closed Turnkey provisioning service? Independent deployment is not documented.
- Does the policy engine expose a time or counter primitive in any private/enterprise tier beyond session TTL?

## 12. Sources

1. Turnkey docs — architecture / concepts overview. `https://docs.turnkey.com/concepts/overview` (accessed 2026-04-18).
2. Turnkey Whitepaper — Principles. `https://whitepaper.turnkey.com/principles` (accessed 2026-04-18). Cites Stellar as an Ed25519-curve chain; documents tiered ecosystem support.
3. Turnkey Whitepaper — Foundations. `https://whitepaper.turnkey.com/foundations` (accessed 2026-04-18). Documents AWS Nitro Enclaves, QuorumOS, PCR/attestation model.
4. Turnkey docs — sub-organization model. `https://docs.turnkey.com/concepts/overview` (accessed 2026-04-18).
5. Turnkey product site. `https://www.turnkey.com/` and `https://www.turnkey.com/embedded-wallets` (accessed 2026-04-18). Framing of "non-custodial."
6. Turnkey GitHub organization. `https://github.com/tkhq` (accessed 2026-04-18). Repository inventory, licences.
7. Turnkey docs — policy language. `https://docs.turnkey.com/concepts/policies/language` (accessed 2026-04-18). Namespace list; absence of Stellar.
8. Turnkey docs — raw payload signing activity. `https://docs.turnkey.com/concepts/policies/language` (`activity.params` discussion, accessed 2026-04-18).
9. Turnkey docs — networks/curves. `https://docs.turnkey.com/networks/overview` and FAQ `https://docs.turnkey.com/faq` (accessed 2026-04-18). Curves: secp256k1, Ed25519, P-256.
10. Turnkey docs — policies overview. `https://docs.turnkey.com/concepts/policies/overview` (accessed 2026-04-18). Effect/consensus/condition; DENY wins precedence.
11. Turnkey docs — root quorum. `https://docs.turnkey.com/concepts/users/root-quorum` (accessed 2026-04-18). Quorum bypass semantics.
12. Turnkey docs — sessions. `https://docs.turnkey.com/authentication/sessions` (accessed 2026-04-18). `expirationSeconds` default 900 s, 10-concurrent-key cap.
13. Turnkey docs — policy language (grammar and operator reference). `https://docs.turnkey.com/concepts/policies/language` (accessed 2026-04-18).
14. Turnkey docs — policy examples. `https://docs.turnkey.com/concepts/policies/examples` (accessed 2026-04-18). Example `consensus`/`condition` triples.
15. Turnkey SDK — React Native session package. `https://www.npmjs.com/package/@turnkey/sdk-react-native` (accessed 2026-04-18). SubtleCrypto/IndexedDB and SecureStorage session storage.
16. QuorumOS repository. `https://github.com/tkhq/qos` (accessed 2026-04-18). AGPL-3.0, release `qos_client-v0.6.1` 2026-04-09.
17. Turnkey engineering post on reproducible builds. `https://quorum.tkhq.xyz/posts/remote-attestations-useless-without-reproducible-builds/` (accessed 2026-04-18). StageX, PCR semantics, EIF reproducibility.
18. Turnkey customer — Moonshot. `https://www.turnkey.com/blog/our-approach-to-multichain-support-at-turnkey` (accessed 2026-04-18). Listed as production user; cited only for adoption signal, not technical claims.
19. Turnkey case study — Spectral Labs. `https://www.turnkey.com/customers/enabling-onchain-ai-agents-with-spectral-labs` (accessed 2026-04-18). AI-agent wallet deployment reference.
20. Turnkey pricing. `https://www.turnkey.com/pricing` (accessed 2026-04-18). Free / Pay-as-you-go / Pro / Enterprise tiers.

## 13. Cross-links

- Compares directly to: **Coinbase Agentic Wallet / CDP Wallets** (same hosted-custody shape), **Safe + modules** (opposite end: on-chain policy, user-held keys), **MoonPay Agents + Open Wallet Standard** (specification vs. proprietary).
- Contributes to requirements around: local policy engine (T2/T4), expiring session credentials (T1), structured `{effect, consensus, condition}` policy triples, root-quorum break-glass (A7), curve-level raw-payload signing as fallback for chains lacking first-class support.
