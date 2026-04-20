# Kraken CLI

**Status:** complete
**Researcher:** research agent
**Last updated:** 2026-04-18
**ID:** kraken-cli
**Category:** crypto-agent

---

## 1. What it is

`krakenfx/kraken-cli` is a Rust single-binary CLI for the Kraken centralised exchange covering spot, tokenised stocks, forex, futures, earn, and funding (151 commands across 13 groups). It also runs as an MCP server over stdio, exposing the same command surface to LLM tool-callers. It is not a wallet: all state and execution live server-side at Kraken; the CLI authenticates REST requests with HMAC-SHA512. Paper-trading engines for spot and futures are client-local simulations over public price feeds ([`kraken-cli/src/paper.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/paper.rs), `futures_paper.rs` @ 506c410).

## 2. Agent interface

CLI invocation pattern: `kraken <command> [args...] -o json 2>/dev/null`; stdout is JSON on success or a JSON error envelope on failure, exit code 0/non-zero (`CLAUDE.md`, `src/main.rs:104` @ 506c410). MCP alternative: `kraken mcp [-s all] [--allow-dangerous]` speaks MCP over stdio via `rmcp` (`src/mcp/server.rs:403-471`). Minimum signed-action sequence (spot buy):

```
export KRAKEN_API_KEY=...; export KRAKEN_API_SECRET=...
kraken order buy BTCUSD 0.001 --type limit --price 50000 --validate -o json   # dry run
kraken order buy BTCUSD 0.001 --type limit --price 50000 -o json              # execute
```

## 3. Custody model

Fully custodial on Kraken. The CLI holds only **API key + secret** (HMAC-SHA512 signing, `src/auth.rs:48-72`); funds and orders live in the user's Kraken exchange account. There is no self-custodial on-chain path: `kraken withdraw <asset> <key> <amount>` only triggers a server-side withdrawal to a pre-configured withdrawal key (`src/commands/funding.rs:110-129`). No wallet keys, no on-chain signing, no chain selection. Paper engines touch only a local JSON state file — simulated money, no keys.

## 4. Policy and approval model

Two mechanisms, both local to the CLI host:
- **Dangerous-tool gate in MCP mode.** 34 commands flagged `dangerous` in `agents/tool-catalog.json` require `"acknowledged": true` in the tool-call arguments, unless started with `--allow-dangerous` (`src/mcp/server.rs:261-289`).
- **`--validate` flag** on order commands forwards `validate=true` to Kraken for server-side dry-run.

No amount limits, counterparty allowlists, time windows, or per-period caps. Kraken-side permissions on the API key (Query Funds, Modify Orders, etc.) are the only durable policy layer. Kraken, not the user, can change server-side limits. Policy does not survive vendor compromise.

## 5. Delegation model

None at the CLI layer. Delegation is expressed through Kraken subaccounts (`kraken subaccount create <username> <email>`) and per-subaccount API keys managed on kraken.com. No cryptographic capability, no scoped signature, no revocation API in the CLI.

## 6. Relevant protocol coverage

Not chain-based. Auth: HMAC-SHA512 with monotonic nanosecond nonce (`src/auth.rs:24-40`). Output envelope:

```
{"error": "auth", "message": "Authentication failed: EAPI:Invalid key"}
```

Error categories (`src/errors.rs:10-20`): `api`, `auth`, `network`, `rate_limit`, `validation`, `config`, `websocket`, `io`, `parse`. Rate-limit envelopes carry `suggestion`, `retryable`, `docs_url` (`src/errors.rs:178-197`). Exit codes: `0` success, `1` any failure (`src/main.rs:22,30,69,104`) — binary, not category-keyed. WebSocket streaming emits NDJSON. Every dispatch path logs structured audit events to stderr (`src/mcp/server.rs:152-157`).

## 7. Licence and adoption signals

- Licence: MIT (`LICENSE` @ 506c410; copyright Payward, Inc.)
- Source available: yes (https://github.com/krakenfx/kraken-cli)
- Last meaningful commit: 506c410 (version bump to 0.3.1)
- GitHub stars: unknown — see §11
- Known production users: Kraken-built; integrations documented for Claude, Cursor, Codex, Copilot, Gemini, Goose (`README.md`)
- Commercial backing: Payward, Inc. (Kraken exchange)
- Ecosystem traction: built-in MCP server; `gemini-extension.json` present in repo

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | no | Kraken holds all keys and funds; CLI holds API credentials only. |
| N2 | Autonomous (no project-operated backend) | no | All REST calls hit `api.kraken.com` / `futures.kraken.com`. |
| N3 | No central server for keys/policy/history | no | All three live on Kraken. |
| N4 | Open source, permissive licence | yes | MIT. |
| N5 | JSON-default output | partial | `-o json` is opt-in; default is `table` (`src/main.rs:14`). |
| N6 | Testnet/mainnet parity | n/a | No chain networks. Paper-mode is structurally distinct: separate commands (`kraken paper`, `kraken futures paper`), responses tagged `"mode":"paper"`/`"futures_paper"` (`src/commands/futures_paper.rs:276`). |

Design reference only; not a deployment model.

## 9. What to adopt

- **Stable error-category envelope** with nine fixed codes and enriched `rate_limit` fields (`suggestion`, `retryable`, `docs_url`). Agents route on category, not on message. Addresses **T3** (typed errors reduce hallucinated retries) and **T2** (machine-routable failure classes). Source: `src/errors.rs:178-197`.
- **Built-in MCP server reusing the CLI dispatch path.** One command tree, two transports; no subprocess wrappers; schemas generated from clap metadata (`src/mcp/server.rs`, `src/mcp/schema.rs`). Serves **A1-A5** simultaneously.
- **Dangerous-tool gate requiring `acknowledged=true`** in MCP mode, with fail-closed argument filtering that drops unknown keys so agents cannot inject `--api-key`/`--api-secret` (`src/mcp/server.rs:265-346`). Addresses **T2** and **T8**.
- **Structured stderr audit log** for every tool call with agent/instance/pid/transport fields and argument *keys only* (never values). Useful for **A1** accountability. Source: `src/mcp/server.rs:143-170`.
- **Structurally distinct paper mode** tagged `"mode":"paper"` in every response. Template for our testnet-vs-mainnet separation against **T10**.
- **Secret-handling discipline:** `--api-secret-stdin`/`--api-secret-file`, 0600 config perms, warning on `--api-secret` flag use (`src/main.rs:43-48`). Addresses **T1**.

## 10. What to avoid

- **Custodial model** fails **N1-N3** and makes all of **T1/T4** Kraken's problem, not the user's — unusable as a deployment target for **A1-A7**.
- **Exit code `1` for every failure class.** Collapses recoverable (`rate_limit`, `network`) and unrecoverable (`auth`, `validation`) into one signal; forces agents to parse JSON from stdout to decide retry policy. Better: category-keyed exit codes. Fails to harden **T3**/**T6** at the shell layer. Source: `src/main.rs:22,30,69,104`.
- **Table output by default.** A human-first default breaks **N5** for agents that forget `-o json`. Source: `src/main.rs:14`.
- **Errors on stdout, not stderr** (`src/output/json.rs:19`). Mixes data and diagnostics on the same stream; agents that pipe stdout to `jq` receive error envelopes indistinguishable from data envelopes except by presence of the `error` key. Complicates **T3** handling.
- **No client-enforced policy.** No amount limits, counterparty allowlists, rate caps. Fails **T2**, **T4**, **T6** entirely when the agent holds the API key.
- **Delegation via shared API keys / subaccounts on the vendor side.** Sub-authority is not cryptographic; revocation requires a Kraken-side action. Fails **T4** as a reference for our delegation story (**A2**).
- **`--allow-dangerous` as a coarse all-or-nothing autonomy switch** with no per-scope, per-amount, or per-counterparty gradation. Habituation risk against **T9**.

## 11. Open questions

- GitHub star count and real-world adoption beyond Kraken's own channels — not derivable from the local clone.
- Whether the MCP `acknowledged=true` pattern generalises to non-binary approvals (amount thresholds, counterparty cohorts). Source shows only the boolean gate.
- `kraken-cli`'s plan for subaccount-level MCP delegation — repo has a `subaccount` service group but no scoped-key workflow exposed through the CLI.

## 12. Sources

- Local: [`kraken-cli/README.md`](https://github.com/krakenfx/kraken-cli/blob/main/README.md) @ 506c410 — product description, command taxonomy, MCP config.
- Local: [`kraken-cli/CLAUDE.md`](https://github.com/krakenfx/kraken-cli/blob/main/CLAUDE.md) @ 506c410 — invocation contract, safety rules.
- Local: [`kraken-cli/src/main.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/main.rs) @ 506c410 — exit-code convention, secret resolution.
- Local: [`kraken-cli/src/errors.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/errors.rs) @ 506c410 — `ErrorCategory`, `to_json_envelope`, rate-limit enrichment.
- Local: [`kraken-cli/src/output/json.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/output/json.rs) @ 506c410 — stdout-only envelope rendering.
- Local: [`kraken-cli/src/mcp/server.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/mcp/server.rs) @ 506c410 — stdio transport via `rmcp`, dangerous-gate, argv sanitisation, audit log.
- Local: [`kraken-cli/src/mcp/mod.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/mcp/mod.rs) @ 506c410 — service-group parsing, streaming exclusions.
- Local: [`kraken-cli/src/auth.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/auth.rs) @ 506c410 — HMAC-SHA512 Spot/Futures signing, nanosecond nonce.
- Local: [`kraken-cli/src/commands/funding.rs`](https://github.com/krakenfx/kraken-cli/blob/main/src/commands/funding.rs) @ 506c410 — `withdraw` semantics (server-side).
- Local: [`kraken-cli/src/commands/futures_paper.rs:276`](https://github.com/krakenfx/kraken-cli/blob/main/src/commands/futures_paper.rs#L276) @ 506c410 — `"mode":"futures_paper"` tagging.
- Local: [`kraken-cli/agents/tool-catalog.json`](https://github.com/krakenfx/kraken-cli/blob/main/agents/tool-catalog.json) @ 506c410 — 151 commands, 34 `dangerous:true`.
- Local: [`kraken-cli/agents/error-catalog.json`](https://github.com/krakenfx/kraken-cli/blob/main/agents/error-catalog.json) @ 506c410 — error envelope reference.
- Local: [`kraken-cli/LICENSE`](https://github.com/krakenfx/kraken-cli/blob/main/LICENSE) @ 506c410 — MIT, Payward, Inc.
- Upstream: https://github.com/krakenfx/kraken-cli (for readers without the local clone).
- Upstream: https://www.kraken.com/kraken-cli (product page; consistent with README).

## 13. Cross-links

- Error-envelope and MCP-server patterns to be considered alongside `research/external/non-crypto/stripe-cli.md`, `gh-cli.md`, `aws-cli.md` (requirement on JSON-default output, structured errors, and built-in agent transport).
- Custody model is a counter-reference to Stellar-native candidates: `meridian-pay.md`, `stellar-mcp.md`, `freighter.md`, `stellar-cli.md` (self-custodial on-chain flows).
- Delegation-via-vendor-subaccount to be contrasted with on-chain smart-account delegation in `research/external/crypto/safe-modules.md`, `turnkey.md`, and Stellar's smart-account candidates.
