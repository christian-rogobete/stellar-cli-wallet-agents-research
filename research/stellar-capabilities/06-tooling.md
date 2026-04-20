# Tooling and APIs

**Status:** complete
**Last updated:** 2026-04-18
**ID:** 06-tooling

---

## 1. What it is

The off-chain surfaces an AI-agent wallet talks to, plus the SDKs that wrap them: the official CLI, Horizon (classic REST), Stellar RPC (Soroban JSON-RPC), the four maintained SDKs (Flutter, iOS/macOS, PHP, KMP), the reference SDKs we borrow patterns from, the testnet faucet, explorer APIs, and the quickstart Docker image for CI. CLI internals are in `research/external/stellar-ecosystem/stellar-cli.md`; signer and key-storage specifics are in `01-accounts.md` and `04-smart-accounts.md`.

## 2. Components

- **`stellar-cli`** — official Rust CLI (`stellar`, alias `soroban`); library crate `soroban_cli` v26.0.0, Apache-2.0. Full taxonomy, non-negotiable check, and agent-gap list in `research/external/stellar-ecosystem/stellar-cli.md`. Gaps relevant here: no MCP or JSON-RPC surface, no policy engine, `--output json` covers only a subset of commands (that file §8).
- **Horizon** — classic REST API ([`js-stellar-sdk/src/horizon/server.ts:27,754-760`](https://github.com/stellar/js-stellar-sdk/blob/master/src/horizon/server.ts:27,754-760)). Per-IP rate limit defaults to 3600 req/hour (`PER_HOUR_RATE_LIMIT`; `developers.stellar.org/docs/data/apis/horizon/api-reference/structure/rate-limiting`, accessed 2026-04-18). SDF runs `horizon.stellar.org` and `horizon-testnet.stellar.org`. SSE via `Accept: text/event-stream` (`…/structure/streaming`, accessed 2026-04-18). Flagged as nearing end-of-life in favour of Stellar RPC + a future Portfolio API (`developers.stellar.org/docs/learn/fundamentals/stellar-stack`, accessed 2026-04-18).
- **Stellar RPC (formerly Soroban RPC)** — JSON-RPC server ([`stellar-rpc/cmd/stellar-rpc/internal/jsonrpc.go:160-271`](https://github.com/stellar/stellar-rpc/blob/main/cmd/stellar-rpc/internal/jsonrpc.go:160-271), tip 2026-04). Twelve methods: `getHealth`, `getEvents`, `getNetwork`, `getVersionInfo`, `getLatestLedger`, `getLedgers`, `getLedgerEntries`, `getTransaction`, `getTransactions`, `sendTransaction`, `simulateTransaction`, `getFeeStats`. No auth in the server (grep `api_key|bearer` in `cmd/stellar-rpc/internal` returns nothing); provider auth is reverse-proxy layered. Default retention ~7 days (`internal/config/options.go:291-298`); `getEvents` caps at 10 000/response, `getTransactions`/`getLedgers` at 200 (`options.go:315-362`). SDF runs free testnet/futurenet; **no SDF mainnet RPC** (`developers.stellar.org/docs/data/apis/rpc/providers`, accessed 2026-04-18).
- **Maintained SDKs (ours)** — `stellar_flutter_sdk` (Dart), `stellar-ios-mac-sdk` (Swift), `stellar-php-sdk` (PHP), `kmp-stellar-sdk` (Kotlin Multiplatform). All cover Horizon + RPC + SEPs with published compatibility matrices ([`stellar_flutter_sdk/README.md`](https://github.com/Soneso/stellar_flutter_sdk/blob/master/README.md), [`kmp-stellar-sdk/README.md`](https://github.com/Soneso/kmp-stellar-sdk/blob/main/README.md), [`stellar-ios-mac-sdk/README.md`](https://github.com/Soneso/stellar-ios-mac-sdk/blob/master/README.md), [`stellar-php-sdk/README.md`](https://github.com/Soneso/stellar-php-sdk/blob/main/README.md)). KMP targets JVM/Android/iOS/macOS/JS/Linux/Windows from one tree; Flutter is the mobile-UX surface; Swift is Apple-native; PHP is server/anchor.
- **Reference SDKs (third-party)** — `js-stellar-base` + `js-stellar-sdk` (SDF), `java-stellar-sdk` and `py-stellar-base` (overcat), `typescript-wallet-sdk` (SDF, higher-level: `Anchor`, `Auth`, `Recovery`, `Uri`, `Watcher` — [`typescript-wallet-sdk/@stellar/typescript-wallet-sdk/src/walletSdk/`](https://github.com/stellar/typescript-wallet-sdk/tree/main/@stellar/typescript-wallet-sdk/src/walletSdk)). Comparators for correctness and API patterns.
- **Friendbot** — HTTP faucet; `GET https://friendbot.stellar.org?addr=<G…>` funds a testnet account with 10 000 XLM ([`js-stellar-sdk/src/horizon/friendbot_builder.ts:4-7`](https://github.com/stellar/js-stellar-sdk/blob/master/src/horizon/friendbot_builder.ts:4-7); `developers.stellar.org/docs/tools/developer-tools/lab/account`, accessed 2026-04-18). Rate-limited, testnet/futurenet only; no mainnet equivalent.
- **Explorers** — StellarExpert (`stellar.expert/openapi.html`, accessed 2026-04-18) exposes an Open API with directory endpoints (tags, known-account categories, fraud-flagged domains) usable for counterparty identity (T5). StellarBeat crawls quorum-set config and per-validator lag (`medium.com/stellarbeatio/stellarbeat-io-lag-detection-and-network-crawler-deep-dive`, accessed 2026-04-18); relevant to validator tier, not tx flows. Stellarchain is secondary; no dependency.
- **Test harness** — `stellar/quickstart` Docker image runs Core + Horizon + RPC + Friendbot in `local`/`testnet`/`futurenet`/`pubnet` mode (`github.com/stellar/quickstart`; `developers.stellar.org/docs/tools/developer-tools/quickstart`, accessed 2026-04-18). Local mode closes a ledger per second; Friendbot on `:8000/friendbot`; RPC on `:8000/rpc`. Testnet/futurenet reset periodically (`developers.stellar.org/docs/networks`, accessed 2026-04-18).

## 3. Relevance to an AI-agent wallet

- **Actors served:** A1, A2, A3, A4, A5, A6, A7 — every actor reads or writes through these surfaces.
- **Non-negotiables touched:** N2 (public endpoints are third-party; wallet must keep working when providers degrade), N5 (JSON native for RPC, enveloped for Horizon, mixed for CLI; SSE parsing load-bearing), N6 (every tool must behave identically on pubnet and testnet).
- **Threats mitigated or created:** T7 (endpoint-configuration attack surface), T10 (network confusion is a tool-default property), T3 (SDK type-safety is first defence against hallucinated fields), T1 (Friendbot keys leaking into prod).

Endpoint configuration is security-critical: a swapped RPC URL defeats simulation-based policy (T7); an ambient network default defeats N6 (T10).

## 4. Agent-specific quirks

- **Horizon SSE under long-running agents.** SSE counts against the 3600 req/hour per-IP budget and goes stale, requiring `Last-Event-ID` reconnects (`stellar.stackexchange.com/questions/1578`, accessed 2026-04-18). A1 watchers need reconnect/backoff logic a browser tab does not.
- **No SDF mainnet RPC.** Moving to pubnet requires a provider or self-host; the wallet cannot hard-code a mainnet default, and multi-endpoint failover is not optional (T7).
- **Retention windows bite long horizons.** ~7-day RPC retention means A5 reconciliation must use Horizon (classic) or client-side archival (Soroban events); `getEvents` returns empty for out-of-window ledgers without distinguishing that from "no events."
- **`simulateTransaction` is trusted but the RPC is not.** Simulation produces the footprint and auth entries we sign; a stale or malicious RPC can return a footprint that succeeds on-chain but off-intent (T7). Remedy: pinned endpoints, cross-checks against a second RPC for high-value actions, or local re-derivation of critical fields.
- **Friendbot is mainnet-absent, testnet-ambient.** `stellar-cli keys fund` and `friendbot()` in every SDK default to the public testnet faucet; a misconfigured agent believing it is funding on pubnet silently hits testnet (T10). The faucet URL is network metadata, not convenience.
- **Mixed-output CLI.** Per `research/external/stellar-ecosystem/stellar-cli.md` §8, only some CLI commands emit JSON. Shelling out for anything else is one text-format tweak away from T3 — the CLI is a reference, not a dependency.
- **StellarExpert is a convenience.** Directory data helps T5 checks but has no SLA or rate-limit contract and opaque licence terms. Policy must not hard-depend on it.
- **Quickstart local = 1 s ledgers.** Fine for CI; fee-stats distribution and sequence-number churn differ from testnet/pubnet (~5 s). Tests passing locally may still surprise in production networks.

## 5. Status on the network

- **Mainnet.** Horizon production but flagged end-of-life; Stellar RPC production via ecosystem providers; no faucet.
- **Testnet/Futurenet.** SDF-operated Horizon, RPC, and Friendbot, free. Periodic resets wipe accounts and contract state (`developers.stellar.org/docs/networks`, accessed 2026-04-18).
- **Roadmap.** Horizon deprecation in favour of Stellar RPC + Portfolio API (`developers.stellar.org/docs/learn/fundamentals/stellar-stack`). No firm dates; classic queries (trades, offers, paths, effects) still need Horizon today. Our maintained SDKs track RPC via per-version compatibility matrices.
- **Known incidents.** Stellar RPC's retention and pagination limits were codified in the 2026-04 tip (`options.go:315-374`); earlier callers that assumed unbounded histories had to update. No mainnet tooling outage is load-bearing for this file.

## 6. Primary documentation

- `research/external/stellar-ecosystem/stellar-cli.md` (internal, 2026-04-18).
- [`stellar-rpc/cmd/stellar-rpc/internal/{jsonrpc.go, methods/, config/options.go}`](https://github.com/stellar/stellar-rpc/blob/main/cmd/stellar-rpc/internal/{jsonrpc.go, methods/, config/options.go}) (tip 2026-04).
- [`js-stellar-sdk/src/{horizon, rpc, friendbot}/`](https://github.com/stellar/js-stellar-sdk/tree/master/src/{horizon, rpc, friendbot}); [`typescript-wallet-sdk/@stellar/typescript-wallet-sdk/src/walletSdk/`](https://github.com/stellar/typescript-wallet-sdk/tree/main/@stellar/typescript-wallet-sdk/src/walletSdk).
- Maintained SDK READMEs and compatibility matrices for the Soneso-maintained client SDKs: [`stellar_flutter_sdk`](https://github.com/Soneso/stellar_flutter_sdk), [`stellar-ios-mac-sdk`](https://github.com/Soneso/stellar-ios-mac-sdk), [`stellar-php-sdk`](https://github.com/Soneso/stellar-php-sdk), [`kmp-stellar-sdk`](https://github.com/Soneso/kmp-stellar-sdk).
- `developers.stellar.org/docs/data/apis/horizon/api-reference/structure/{rate-limiting, streaming}`; `…/data/apis/rpc/providers`; `…/tools/developer-tools/quickstart`; `…/networks`; `…/learn/fundamentals/stellar-stack` (all accessed 2026-04-18).
- `stellar.expert/openapi.html`; `github.com/stellar-expert/stellar-expert-explorer`; `github.com/stellar/quickstart` (accessed 2026-04-18).

## 7. Known gaps or pain points

- **No first-party MCP or wallet-level typed IPC.** Stellar RPC is the only structured surface SDF ships; any agent surface for the *wallet* must be designed from scratch. x402-MCP is a separate SDF track, not a wallet protocol.
- **Provider auth is out-of-band.** Stellar RPC has no built-in per-call auth; providers layer headers via reverse proxies. Config must accept opaque header maps per endpoint.
- **Horizon end-of-life is soft.** Documented without dates or a Portfolio-API spec; commitment to RPC-only is premature.
- **CLI JSON coverage.** Per external CLI file §8; anything outside the listed commands is screen-scrape territory.
- **Explorer licensing.** StellarExpert directory data is "publicly available for developers and users, free of charge" (`github.com/stellar-expert/stellar-expert-explorer` README, accessed 2026-04-18) but responses carry no machine-readable licence; bundled caches need legal review.
- **Quickstart has a fixed root account.** Explicitly "not suitable for any production use" (`developers.stellar.org/docs/tools/quickstart/network-modes`, accessed 2026-04-18).
- **Reference-implementation language is not picked here.** KMP has the widest platform reach; Flutter and iOS/macOS win mobile UX; PHP is the server/anchor choice. The CLI-first vs. agent-native decision dictates the fit.

## 8. Candidate requirements surfaced

- `REQ-cfg-endpoints-multi`: MUST accept per-network lists of RPC and Horizon endpoints (primary + fallbacks). MUST. Source: `06-tooling §4 (No SDF mainnet RPC; T7)`.
- `REQ-cfg-endpoint-auth`: MUST support opaque HTTP-header maps per endpoint for provider API keys. MUST. Source: `06-tooling §7 (Provider auth out-of-band)`.
- `REQ-cfg-endpoint-pinning`: SHOULD support pinned TLS certificates for user-operated endpoints. SHOULD. Source: `06-tooling §4 (simulateTransaction trust; T7)`.
- `REQ-cfg-crosscheck`: SHOULD cross-check high-value transactions against a second RPC before signing. SHOULD. Source: `06-tooling §4 (simulateTransaction trust; T7)`.
- `REQ-cfg-network-explicit`: MUST require the network (passphrase + endpoint set) per call; no ambient default. MUST. Source: `06-tooling §4 (Friendbot default; T10)`.
- `REQ-ux-friendbot-scoped`: MUST restrict faucet calls to testnet/futurenet identities and error on pubnet. MUST. Source: `06-tooling §4 (Friendbot mainnet-absent)`.
- `REQ-cfg-sse-reconnect`: MUST implement SSE reconnect with `Last-Event-ID` and exponential backoff. MUST. Source: `06-tooling §4 (SSE stale streams)`.
- `REQ-cfg-retention-aware`: MUST distinguish "no events in window" from "out of retention window." MUST. Source: `06-tooling §4 (Retention window)`.
- `REQ-cfg-quickstart-support`: CI harness SHOULD support `stellar/quickstart` local mode alongside testnet. SHOULD. Source: `06-tooling §2 (Test harness)`.
- `REQ-ux-explorer-optional`: Explorer data MUST be advisory; policy MUST NOT depend on explorer availability. MUST. Source: `06-tooling §7 (Explorer licensing)`.
- `REQ-cfg-sdk-parity`: Reference implementation MUST build on a maintained SDK with current, published compatibility matrices. MUST. Source: `06-tooling §2 (Maintained SDKs)`.
- `REQ-ux-horizon-deprecation`: Data-access layer SHOULD abstract Horizon vs. RPC so the migration is client-layer-only. SHOULD. Source: `06-tooling §5 (Horizon end-of-life)`.

## 9. Cross-links

- Related capability files: `01-accounts.md` (SDK signer backends), `03-soroban.md` (simulation, events, auth — RPC-backed), `04-smart-accounts.md` (contract-address credentials; CLI limit in external CLI §5), `05-seps.md` (SEP-10/-24 client code in `typescript-wallet-sdk` `Auth`, mirrored in our SDKs), `08-infra-ops.md` (rate limits, retries, LaunchTube alternatives).
- External candidates touching this capability: `stellar-ecosystem/stellar-cli.md`, `stellar-ecosystem/stellar-mcp-server.md`, `stellar-ecosystem/freighter-walletkit.md`, `stellar-ecosystem/meridian-pay.md`.
- Decision files that will cite this entry: `analysis/03-requirements.md` (all REQs), `analysis/06-option-a-cli-first.md` (`soroban_cli` library reuse), `analysis/07-option-b-agent-native.md` (SDK under the agent-native surface), `analysis/10-recommendation.md` (endpoint-configuration posture).
