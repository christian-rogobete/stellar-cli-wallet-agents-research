# LaunchTube

**Status:** complete
**Researcher:** research agent (Opus 4.7 1M)
**Last updated:** 2026-04-18
**ID:** launchtube
**Category:** crypto-stellar

---

## 1. What it is

HTTP submission service for Soroban `invokeHostFunction` ops, authored by kalepail (Tyler van der Hoeven), hosted by SDF at `launchtube.xyz` / `testnet.launchtube.xyz`. Accepts a caller-signed Soroban op, draws a sequence number from a pool of server-funded channel accounts, wraps the inner envelope in a fee-bump signed by a server-held `FUND_SK`, submits via RPC. Single Cloudflare Worker + Durable Objects [S1, `wrangler.toml`]. **Now legacy**: README says "superseded by the OpenZeppelin Relayer with the Channels Plugin"; Stellar docs state SDF is discontinuing it for lack of "maturity, scalability and auditing" [S1, `README.md:3-48`; S2].

## 2. Agent interface

`POST /` with bearer JWT, `x-www-form-urlencoded`. Minimum signed action: (1) `GET /gen` (testnet) or Discord request (mainnet) → JWT; (2) sign Soroban op locally; (3) `POST /` with `xdr=<envelope>` or `func=<HostFunction XDR>` + `auth=<[SorobanAuthorizationEntry XDR]>`; (4) server rebuilds tx against a pool sequence account, simulates, signs with sequence keypair, fee-bumps with `FUND_SK`, submits, polls ≤30 s, returns JSON [S1, `src/api/launch.ts:96-336`]. Soroban-ops only — classic payments, SEPs, `changeTrust` rejected [S1, `src/api/launch.ts:116-122`].

## 3. Custody model

- **Caller keeps user keys.** Schema is `xdr`, `func`, `auth` — no secret-key field [S1, `src/api/launch.ts:23-35`]. Only signed XDR crosses the wire.
- **Operator holds two key classes:** (a) master `FUND_SK` paying all fee-bumps [S1, `src/api/launch.ts:296-304`]; (b) sequence-account keypairs in Durable-Object storage, derived deterministically as `SHA-256(FUND_SK || index)` [S1, `src/sequencer.ts:62-143`].
- **What the operator sees:** plaintext inner tx, auth entries, memo (propagated via `memo: tx?.memo` [S1, `src/api/launch.ts:210`]), contract address, function name, args, caller source account, and JWT `sub` — all logged against the tx hash in a SQL table SDF queries via an unvalidated `POST /sql` admin endpoint [S1, `src/api/sql.ts:21-24`; `README.md:292-331`]. Operator correlates txs to bearer tokens.

## 4. Policy and approval model

None at LaunchTube. Structural audits only: `sorobanCredentialsSourceAccount` cannot reuse the sequence account [S1, `src/api/launch.ts:144-172`]; `timeBounds.maxTime` ≤ 30 s; inner fee ≤ resourceFee + 201 [S1, `src/api/launch.ts:250-262`]. No amount caps, counterparty allowlists, asset scoping, rate limits. KALE has a hard-coded min-fee special case [S1, `src/api/launch.ts:274-279`].

## 5. Delegation model

None. JWTs are bearer tokens with pre-paid credit balances; progressive spend (eager → bid → final, refund on success) [S1, `README.md:86-94`; `src/credits.ts`]. No scope, no counterparty allowlist, no on-chain revocation — only `DELETE /:sub` [S1, `src/api/token-delete.ts`].

## 6. Relevant protocol coverage

Soroban `invokeHostFunction` + `createContractV2` only [S1, `src/api/launch.ts:138-141`]. Sequence-pool: random pick of up to 100 entries, in-flight keys moved to `field:` namespace, returned on success/failure, GC'd after 5 min [S1, `src/sequencer.ts:30-60`; `src/common.ts:145-162`]. Fee-bump wrapper pays XLM for the caller. Cloudflare Workers + Durable Objects + KV.

## 7. Licence and adoption signals

- **Licence: none.** No `LICENSE` file; `package.json` is `"private": true` with no `license` field [S1]. Source-available, not open-source.
- Repo `github.com/stellar/launchtube` (moved from `kalepail/launchtube`). Last commit 2026-01-13 (`1821ebd`, deprecation banner).
- Known consumers: Meridian Pay (mandatory fee-bump), Stellar MCP server (hard dep), Passkey Kit [S3], KALE miners [S4].
- Operator: SDF, single deployment. README warns "experimental … no guarantees … do not use for mission-critical production services" [S1, `README.md:60-61`]. `TODO we've lost a couple sequences somehow` comment in production [S1, `src/sequencer.ts:56-57`].

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | **yes** | User keys never transit; only signed XDR [S1, `src/api/launch.ts:23-35`]. |
| N2 | Autonomous | **no (hosted) / partial (self-hosted)** | Every write hits SDF's hosted service. Self-hosting is technically possible (`wrangler deploy`) but requires owning a Cloudflare account + funded `FUND_SK`. |
| N3 | No central server for keys/policy/history | **no** | Operator holds `FUND_SK` + sequence-pool keypairs, logs every tx with JWT `sub` in a queryable SQL table [S1, `src/sequencer.ts:62-77`; `src/api/sql.ts`]. |
| N4 | Open source, permissive licence | **no** | No LICENSE; `package.json private:true`. Cannot fork-and-ship. |
| N5 | JSON-default output | yes | RPC JSON passed through. |
| N6 | Testnet/mainnet parity | yes | Separate deployments, same API [S1, `wrangler.toml`]. |

**Verdict: fails N2 / N3 / N4 as hosted.** Self-hosting clears N2/N3 only for a single operator; N4 still fails. Disqualification is largely **inherent** (operator holds keys, logs envelopes, no licence), not situational.

## 9. What to adopt

- **Sequence-pool with deterministic derivation** [S1, `src/sequencer.ts:62-143`] — `SHA-256(master || index)` seeds let us lazily rebuild an in-process channel-account pool from the agent's mnemonic. Primitive for A1/A2 parallel submission; pool lives on the user's host.
- **Credit-as-capability accounting** [S1, `src/credits.ts`] — pre-paid balance with progressive spend (eager → bid → final, refund unused). Maps to our local per-period budget for A1/A3.
- **Single-op typing + auth-entry audit** [S1, `src/api/launch.ts:116-172`] — reject anything not one `invokeHostFunction`; verify simulation auth matches submitted auth. T3.
- **Short `timeBounds.maxTime` (30 s default)** [S1, `src/api/launch.ts:260-262`] — narrow window for agent-submitted txs.
- **Fee-bump as submission primitive, not custody primitive** — caller signs authorisation, local fee-bumper pays XLM. Decouples policy from fee funding (A1/A5).

## 10. What to avoid

- **Third-party submitter seeing inner envelopes + identifiers** — the root cause of Meridian Pay and Stellar MCP failing N2/N3 (`meridian-pay.md` §8; `stellar-mcp-server.md` §8). No hard dependency on any such service. N2/N3; widens T7.
- **Server-held master funding signer** [S1, `src/sequencer.ts:66`, `src/api/launch.ts:296`] — one `FUND_SK` compromise drains every pool account. T1.
- **Unvalidated `POST /sql` admin endpoint** [S1, `src/api/sql.ts:21-24`] — anti-pattern against T8.
- **Bearer-JWT capability with no scope** — token exfil = full credit drain. T1/T4. Any equivalent must be scoped to counterparty + period.
- **No licence** — cannot fork; any adopted idea is clean-room. N4.
- **Building A1/A2 submission on a service SDF labels "do not use for mission-critical production"** [S1, `README.md:60-61`] and is actively discontinuing [S2].

### In-process alternatives for parallel submission (A1/A2)

1. **Local channel-account pool** — SEP-5 HD-derived ed25519 keys, funded from agent master, round-robin. Same semantics as LaunchTube's pool, zero operator. Classic + Soroban. Cost: 1 XLM min-balance per channel.
2. **Per-agent subaccounts** — each sub-agent gets its own G-account, funded via Merkle-distributor (`meridian-pay.md` §9). Best where per-sub-agent revocation matters.
3. **Smart-account policy delegation** — C-account (OZ Policy trait); sub-agents sign auth entries under bounded scope. Soroban nonces → no sequence coordination. Best for on-chain-revocable A2 (T4).
4. **Self-hosted LaunchTube fork** — feasible but needs Cloudflare + funded `FUND_SK` + re-licence. Not a path for a self-custodial wallet.

## 11. Open questions

- Does the OpenZeppelin Relayer Channels Plugin (AGPL-3.0, [S2]) offer a different trust model self-hosted, or does it log inner envelopes the same way? Warrants its own tier-2 file; AGPL likely clashes with N4 as Safe's LGPL does.
- Does any consumer (Passkey Kit, Meridian Pay) ship a documented off-path that bypasses LaunchTube? Passkey Kit says "not a requirement" [S3] but ships no alternative.
- Is SDF planning to lift the sequence-pool pattern into Stellar RPC as a native feature? Would obviate both LaunchTube and OZ Relayer.

## 12. Sources

- [S1] Local clone `/tmp/launchtube-tmp/` from `github.com/stellar/launchtube` @ `1821ebdd95829e8d023f5291baaa9df424af489d` (2026-01-13) — `README.md`, `src/api/launch.ts`, `src/api/launch-v2.ts`, `src/api/sql.ts`, `src/api/token-delete.ts`, `src/sequencer.ts`, `src/common.ts`, `src/credits.ts`, `src/helpers.ts`, `wrangler.toml`, `package.json`. Accessed 2026-04-18.
- [S2] Stellar Docs, "OpenZeppelin Relayer." `https://developers.stellar.org/docs/tools/openzeppelin-relayer` — accessed 2026-04-18. SDF deprecation quote: "Launchtube does not have the maturity, scalability and auditing as OpenZeppelin's Relayer service, which is why the Stellar Development Foundation is discontinuing the Launchtube service."
- [S3] `github.com/kalepail/passkey-kit` README. Accessed 2026-04-18.
- [S4] `github.com/kalepail/KALE-sc` README and `github.com/FredericRezeau/kale-miner` config. Accessed 2026-04-18.

## 13. Cross-links

- Counter-pattern to `research/external/stellar-ecosystem/meridian-pay.md` (wallet-backend fee-bump is the same centralised-submitter failure).
- Counter-pattern to `research/external/stellar-ecosystem/stellar-mcp-server.md` (its LaunchTube dep is the direct cause of its N2/N3 failures).
- Feeds `research/stellar-capabilities/08-infra-ops.md` — in-process channel-pool + smart-account-delegation alternatives.
- Feeds `analysis/01-actor-model.md` A1/A2 (parallel submission) and `analysis/02-threat-model.md` T1/T7 (submitter trust + correlation).
