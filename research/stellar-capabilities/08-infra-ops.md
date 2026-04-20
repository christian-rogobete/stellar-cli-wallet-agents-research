# Operational Infrastructure

**Status:** complete
**Last updated:** 2026-04-18
**ID:** 08-infra-ops

---

## 1. What it is

Plumbing that turns a locally-signed transaction into a recorded ledger effect: who pays XLM fees, how sequence numbers are managed under concurrency, how to retry a timed-out submission without double-spending, how to survive public-endpoint rate limits, what to record for audit, and — since 2026-Q1 — which submitter to target now that LaunchTube is going away. Custody, SEPs, and policy live in sibling capability files.

## 2. Components

- **CAP-0033 Sponsored Reserves.** `BeginSponsoringFutureReservesOp` / `EndSponsoringFutureReservesOp` sandwich in one transaction; between them, reserves for new accounts, trustlines, signers, and claimable balances are charged to the sponsor. `RevokeSponsorshipOp` moves existing reserves. [`stellar-protocol/core/cap-0033.md:14-56`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0033.md:14-56).
- **CAP-0015 Fee-Bump Transactions.** `FeeBumpTransaction` wraps an inner envelope and is signed by a different `feeSource`; inner signatures and seq num preserved. `ENVELOPE_TYPE_TX_FEE_BUMP = 5`. [`stellar-protocol/core/cap-0015.md:14-76`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0015.md:14-76).
- **Operator SDK surfaces.** Flutter: `FeeBumpTransaction`, `FeeBumpTransactionBuilder` (`/Users/chris/projects/Stellar/stellar_flutter_sdk/lib/src/transaction.dart:697-825`); sponsorship XDR types first-class (`.../xdr_sponsorship_descriptor.dart`, `.../xdr_revoke_sponsorship_*.dart`). iOS / PHP / KMP SDKs mirror.
- **`stellar-cli` ops surface.** `stellar tx new begin-sponsoring-future-reserves --sponsored-id <G...>`, `end-sponsoring-future-reserves`, `revoke-sponsorship` ([`stellar-cli/cmd/soroban-cli/src/commands/tx/new/{begin,end}_sponsoring_future_reserves.rs`](https://github.com/stellar/stellar-cli/blob/main/cmd/soroban-cli/src/commands/tx/new/{begin,end}_sponsoring_future_reserves.rs), `.../revoke_sponsorship.rs`). `stellar tx update sequence-number next` rewrites seq num via RPC (`.../update/sequence_number/next.rs:44-74`). `stellar keys generate|add|fund|use` manage file or OS-keyring identities ([`stellar-cli/FULL_HELP_DOCS.md:1109-1195`](https://github.com/stellar/stellar-cli/blob/main/FULL_HELP_DOCS.md:1109-1195)).
- **Soroban `sendTransaction` / `getTransaction`.** Submission is async; statuses `PENDING | DUPLICATE | TRY_AGAIN_LATER | ERROR`; poll `getTransaction` for terminal outcome. Default RPC retention 24 h, up to 7 d self-hosted [docs.stellar.org/docs/data/apis/rpc/api-reference/methods/getTransaction, 2026-04-18].
- **Horizon rate limiting.** Per-IP `PER_HOUR_RATE_LIMIT`, default 3600 req/hr; public operators historically 7200-17200 req/hr; HTTP 429 with `X-Ratelimit-*` + `Retry-After` [docs.stellar.org/docs/data/apis/horizon/api-reference/structure/rate-limiting, 2026-04-18].
- **Soroban nonces.** `SorobanAddressCredentials.nonce` is `int64`, consumed only when its auth entry is used, valid until `signatureExpirationLedger` ([`stellar-protocol/core/cap-0046-11.md:148-456`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-11.md:148-456)). Decouples smart-account auth from classic sequence numbers.
- **OpenZeppelin Relayer (Stellar Channels Service).** Self-hostable Rust service wrapping fee-bump + channel-account submission; hosted at `channels.openzeppelin.com` [developers.stellar.org/docs/tools/openzeppelin-relayer, 2026-04-18]. `@openzeppelin/relayer-plugin-channels` is the TypeScript driver [github.com/OpenZeppelin/relayer-plugin-channels, 2026-04-18].
- **LaunchTube (legacy).** Hosted fee-bump + channel-pool front-end for Soroban ops; SDF end-of-life announced (`research/external/_tier-2/stellar-ecosystem/launchtube.md` §1, §7).

## 3. Relevance to an AI-agent wallet

- **Actors served:** A1, A2, A3, A5.
- **Non-negotiables touched:** N1, N2, N3, N4, N5, N6.
- **Threats mitigated or created:** T1, T4, T6, T7, T8, T10 (mitigated); T8 raised by any remote submitter dependency.

Submission infrastructure is the load-bearing choice for an autonomous wallet: it decides whether A1/A2 submit at usable throughput without a project-operated backend (N2), whether A5 is idempotent under retry, and whether A3 pays sub-cent fees without a funding ceremony. LaunchTube's retirement reframes this area from "pick a hosted submitter" to "ship a first-class in-process submitter." N3 rules out any remote submitter that logs inner envelopes — hosted LaunchTube, and under N4 also hosted OZ Relayer as a deployment dependency.

## 4. Agent-specific quirks

- **Seq nums are monotonic per source account.** Parallel submissions from one account collide as `tx_bad_seq`. A1/A2 concurrent submitters need one-at-a-time queueing, a channel-account pool with fee-bump wrapper, or per-agent G-accounts. `fetch-seq-then-sign` races are the modal bug.
- **Fee-bumps do not parallelise.** Inner tx still gated by its own seq num; fee-bump only lets a different account pay fees and raises `minFee`. Parallelism comes from distinct source accounts on inner txs.
- **Sponsored reserves sandwich in one tx.** `Begin` and `End` sit in the same tx; every sponsored ledger-entry change between them. Iterative op-appending breaks this; collect intent first, lay out once.
- **Soroban nonces are per-(address, signature), not per-account.** Smart-account signers can issue parallel auth entries with different nonces; outer txs still need distinct seq nums. Structural advantage of smart-account delegation over classic multisig for A2.
- **`sendTransaction` is fire-and-forget.** Statuses do not tell whether the tx was included; poll `getTransaction` within the retention window.
- **Hash is the natural idempotency key.** Signed envelope hash is deterministic; `DUPLICATE` on resubmit is built-in de-dup. Maps onto Stripe-CLI `Idempotency-Key` semantics (`research/external/non-crypto/stripe-cli.md` §6): persist hash + last-known status, resubmit the same envelope.
- **Unit confusions (T3) reappear.** Fees in stroops; `inner.fee + feeBump.fee` differs from classic. Expose typed units at the wallet surface.
- **Network is in the tx hash.** Cannot replay cross-network, but the agent can still draft the wrong one (T10). Require network explicitly at build time.

## 5. Status on the network

- Mainnet: sponsored reserves (CAP-0033, Final, protocol 14/15), fee-bumps (CAP-0015, Final, protocol 13), Soroban nonces (CAP-0046-11) — production. OZ Relayer: production-ready self-host or hosted as of 2026-Q1.
- Testnet: parity; `testnet.launchtube.xyz` remains during the sunset window.
- Roadmap: no CAP in flight changing these primitives materially. Open question whether SDF will lift channel-account semantics into stellar-rpc (`research/external/_tier-2/stellar-ecosystem/launchtube.md` §11).
- Known incidents: LaunchTube deprecation is the only active change.

## 6. Primary documentation

All accessed 2026-04-18.

- CAP-0033 Sponsored Reserve: [`stellar-protocol/core/cap-0033.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0033.md).
- CAP-0015 Fee-Bump Transactions: [`stellar-protocol/core/cap-0015.md`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0015.md).
- CAP-0046-11 Soroban auth / nonces: [`stellar-protocol/core/cap-0046-11.md:148-456`](https://github.com/stellar/stellar-protocol/blob/master/core/cap-0046-11.md:148-456).
- stellar-cli sponsorship / seq-num helpers: [`stellar-cli/cmd/soroban-cli/src/commands/tx/new/{begin,end}_sponsoring_future_reserves.rs`](https://github.com/stellar/stellar-cli/blob/main/cmd/soroban-cli/src/commands/tx/new/{begin,end}_sponsoring_future_reserves.rs), `.../update/sequence_number/next.rs`.
- Flutter SDK fee-bump: `/Users/chris/projects/Stellar/stellar_flutter_sdk/lib/src/transaction.dart:697-825`.
- Horizon rate-limiting: `developers.stellar.org/docs/data/apis/horizon/api-reference/structure/rate-limiting`.
- Soroban RPC: `developers.stellar.org/docs/data/apis/rpc/api-reference/methods/{sendTransaction,getTransaction}`.
- OZ Relayer on Stellar: `developers.stellar.org/docs/tools/openzeppelin-relayer`. OZ docs: `docs.openzeppelin.com/relayer` ("licensed under the GNU Affero General Public License v3.0"). Repo: `github.com/OpenZeppelin/openzeppelin-relayer` (AGPL-3.0).
- OZ Channels Plugin repo: `github.com/OpenZeppelin/relayer-plugin-channels` (`LICENSE` AGPL-3.0 per GitHub API + verbatim file).
- LaunchTube analysis: `research/external/_tier-2/stellar-ecosystem/launchtube.md`.

## 7. Known gaps or pain points

- **OZ Relayer Channels Plugin licence — resolved AGPL-3.0, one residual inconsistency.** The plugin's `LICENSE` file is verbatim GNU AGPL v3; GitHub's auto-detected licence is `agpl-3.0`; main `openzeppelin-relayer` repo and `@openzeppelin/relayer-sdk` both declare AGPL-3.0-or-later. But the plugin's `README.md` "License" section and `package.json` `"license"` field both state `MIT` — a vendor-side defect worth flagging before adoption; `LICENSE` controls. **Conclusion:** treat as AGPL-3.0. Per OZ's AGPL FAQ [openzeppelin.com/agpl-license, 2026-04-18], internal use and self-hosting for one's own agent are within licence; publish-source triggers apply only on distribution or third-party network access. Against N4: acceptable as a user-self-hosted optional backend; poor fit as a *mandatory* wallet-shipped dependency. Resolves the gap in `analysis/00-context.md` §5.4.
- **Hosted OZ Relayer fails N2/N3.** Same shape as LaunchTube: hosted Channels Service sees inner envelopes, operator holds fund-relayer key, logs per-relayer-id usage.
- **Sequence-pool minimum-balance cost.** In-process pool of N channel accounts costs N × 1 XLM base reserve plus trustline reserves. Sponsored-reserve helps only for ledger entries, not for the base-account reserve of the channel account itself.
- **Idempotent retry under seq-number semantics is not safe by default.** Sign tx-A with seq N, time out, fetch seq, sign tx-B with seq N+1 — resubmitting tx-A after a transient RPC blip can re-apply it. Correct pattern: tight `timeBounds.maxTime`, persist hash before submit, poll `getTransaction(hash)`, replace only after the prior envelope's timeBound has passed.
- **`getTransaction` 24-hour retention.** Retry loops blocking for hours lose confirmation after the window closes; persistent receipts on the agent host cover this.
- **Horizon rate limits vary in practice.** Public Horizon has run 3600-17200 req/hr; streaming consumers hit this quickly. Retry-After compliance + exponential backoff + user-configurable paid endpoints required for A1/A2 (T7).
- **MPP channel mode as an off-chain submission pattern.** MPP's channel (session) intent is a separate submission model: the client does not submit transactions per request; it signs off-chain cumulative commitments that the server settles on-chain at close time. This is orthogonal to the sequence-pool + fee-bump paths above and serves the same actor profile (A1, A3 at high frequency). Wallet implications: channel-close transactions are submitted by the server, not the wallet; the wallet's audit record must correlate a local sequence of signed commitments to a single eventual on-chain close. No formal spec yet; see `10-mpp.md` §2.2 and §7.

## 8. Candidate requirements surfaced

- `REQ-perf-seq-pool`: in-process channel-account pool, SEP-5 HD-derived, so A1/A2 submit in parallel without a hosted submitter. **MUST**. `launchtube.md` §10; A1, A2, N2, N3.
- `REQ-perf-fee-bump`: every wallet-initiated tx supports local fee-bump wrapping with a configurable fee-payer. **MUST**. CAP-0015; A1, A3, A5.
- `REQ-perf-sponsored-reserves`: Begin/End/Revoke sponsorship as first-class CLI ops matching stellar-cli's shape. **MUST**. CAP-0033; A2, A3.
- `REQ-perf-soroban-smart-account-delegation`: smart-account policy delegation using Soroban nonces for parallel auth without classic-seq coordination. **SHOULD**. CAP-0046-11; A2, T4.
- `REQ-audit-submit-idempotent`: submit path safe to retry by tx-hash; wallet persists envelope hash before submit, polls `getTransaction(hash)`, surfaces `DUPLICATE` as success. **MUST**. `stripe-cli.md` §9; A1, A5, T6.
- `REQ-cfg-endpoints`: every network-selecting command accepts user-provided Horizon / Soroban RPC endpoints; no project-operated default. **MUST**. N2, N3, T7.
- `REQ-cfg-rate-limit`: retries HTTP 429 using `Retry-After` + capped exponential backoff; per-endpoint max concurrency configurable. **SHOULD**. Horizon rate-limiting docs; A1, A6.
- `REQ-cfg-relayer-opt-in`: OZ Relayer / Channels Service supported as opt-in only; requires an explicit user endpoint and warns the operator sees the inner envelope. **NICE**. §7; N2, N3, N4.
- `REQ-audit-trail`: every signature, policy decision, submitted hash, and receipt appended to a local hash-chained tamper-evident log. **MUST**. `analysis/02-threat-model.md` T8; A1, A2, A5.
- `REQ-perf-timebounds`: outer txs default to short `timeBounds.maxTime` (30 s unless overridden) so timed-out retries are safe by construction. **SHOULD**. `launchtube.md` §9; A1, A5, T6.

**Recommended in-process alternative for v1.** Ship `REQ-perf-seq-pool` (local SEP-5-derived channel-account pool) as the default submission primitive, with `REQ-perf-fee-bump` wrapping every outer tx. Per-agent subaccounts and smart-account policy delegation are scale-up paths, not v1 defaults. The channel-account pool replicates LaunchTube's throughput win, lives entirely on the user's host, and does not require a Soroban contract per wallet. Treat OZ Relayer / Channels Plugin as optional (`REQ-cfg-relayer-opt-in`), not a dependency.

## 9. Cross-links

- Related capability files: `03-soroban.md` (auth entries, simulate-before-submit), `04-assets-and-payments.md` (classic ops paying for sponsorship), `05-seps.md` (SEP-10 rate considerations), `07-smart-accounts.md` (smart-account policy delegation).
- External candidates: `research/external/_tier-2/stellar-ecosystem/launchtube.md` (primary input), `research/external/stellar-ecosystem/meridian-pay.md` §6 (wallet-backend fee-bump precedent), `research/external/non-crypto/stripe-cli.md` §9 (idempotency pattern), `research/external/stellar-ecosystem/stellar-mcp-server.md` (consumer of deprecated LaunchTube dep).
- Decision files that will cite this entry: `analysis/00-context.md` §5.4 (OZ Relayer licence resolution), `analysis/01-actor-model.md` A1/A2/A5, `analysis/02-threat-model.md` T1/T7/T8, `analysis/03-requirements.md` (candidate queue).
