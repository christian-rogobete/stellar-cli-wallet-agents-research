# Trust Wallet Agent Kit (TWAK)

**Status:** complete
**Researcher:** agent (Opus 4.7, 1M)
**Last updated:** 2026-04-18
**ID:** trust-wallet-agent-kit
**Category:** crypto-agent

---

## 1. What it is

TWAK is a developer toolkit from Trust Wallet (the Binance-affiliated, self-custodial consumer wallet with ~220M installs) that lets AI agents execute on-chain operations on 25+ chains. It ships two artifacts: the `twak` CLI (`@trustwallet/cli` on npm, v0.9.0 at time of research [S5]) which doubles as an MCP stdio server and a REST server, and the `trustwallet/tw-agent-skills` skill-marketplace repo [S1] that supplies "skill" documentation for coding agents (Claude Code, Cursor, Codex, Windsurf, Copilot, Cline, opencode, roo). TWAK operates in two modes: an "agent wallet" mode where the CLI holds an HD wallet on disk, and a "WalletConnect" mode where the CLI proposes transactions to a user's existing Trust Wallet mobile app for approval [S2].

## 2. Agent interface

CLI, MCP (stdio), and REST HTTP are all exposed by the same binary. Install `npm install -g @trustwallet/cli`; authenticate with HMAC-SHA256 credentials from `portal.trustwallet.com` (`TWAK_ACCESS_ID`, `TWAK_HMAC_SECRET`); run `twak init` to persist credentials (`~/.twak/credentials.json` mode 0600, HMAC secret mirrored to OS keychain) [S1 `skills/wallet/references/setup.md:54-61`].

Minimum signed-action sequence (agent wallet mode):

```
twak wallet create --password <pw> --json
twak wallet balance --chain ethereum --json
twak transfer --to vitalik.eth --amount 0.01 --token c60 --json
```

Every command accepts `--json`. Agents typically connect via `twak serve` (MCP stdio) — see §6. Asset IDs use a compact SLIP-44 form `c{coinId}[_t{contract}]` [S1 `skills/api/references/setup.md:146-151`].

## 3. Custody model

**Mode A — Agent wallet.** HD (BIP39) wallet created by the CLI; mnemonic at `~/.twak/wallet.json`, AES-256-GCM encrypted with a user password. Password is auto-resolved from OS keychain (macOS Keychain / Linux Secret Service / Windows Credential Manager) unless `--password` is passed or `TWAK_WALLET_PASSWORD` is set [S1 `skills/wallet/references/setup.md:100-106`, `wallet.md:56-59`]. Signing happens locally via bundled WalletCore WASM (`package/trust-sdk/wallet-core.wasm`). Keys never leave the host. Custody rests with whoever controls the machine's file system and keychain.

**Mode B — WalletConnect.** `twak wallet connect --project-id <id>` opens a WalletConnect v2 session to the user's Trust Wallet mobile app [S1 `skills/wallet/references/wallet.md:61-73`]. The CLI holds no keys in this mode — it is a proposer. Each on-chain action becomes a `wc_sessionRequest` (tag 1108, 15-minute TTL) that prompts the user in their Trust Wallet app [S5 bundle code — standard WalletConnect v2 relay constants]. Custody remains with the Trust Wallet mobile app.

## 4. Policy and approval model

Policy is expressed per-command, not as a persistent rule set:

- `transfer --max-usd <n>` — rejects a transfer above the USD cap (default 10000); `--skip-safety-check` bypasses [S1 `skills/wallet/references/send.md:68-69`].
- `transfer --confirm-to <address>` — pins the ENS-resolved address; mismatch aborts the tx [S1 `send.md:67`].
- `swap --slippage <pct>` (default 1%); `--quote-only` dry-runs [S1 `swap.md:49-52`].
- `erc20 approve --amount unlimited --confirm-unlimited` — unlimited ERC-20 approvals require an explicit extra flag [S1 `erc20.md:34-37`].
- `x402 request --max-payment <amount>` (default 0.01) with interactive confirm unless `--yes` [S1 `x402.md:28-48`].

For automations: `automate add` accepts `--max-runs` and `--expires`; automations only execute while `twak watch` is running (a persistent foreground poller) [S1 `automations.md:63-89`]. The `serve` REST mode supports `--auto-lock <minutes>` to evict the wallet password from memory and `--x402` to gate the REST endpoints with on-chain payment [S5 `registerServeCommand`].

No persistent signed-policy file, no counterparty allowlist primitive, no per-period USD ceiling, no per-asset cap, no minimum-reserve guard. Limits are per-invocation CLI flags an agent must pass each call — the agent is trusted to apply them. This fails T2 and T6 as hard defences.

## 5. Delegation model

TWAK has no delegation primitive. There is one key (or one WalletConnect session) per CLI install. Sub-agent scoping, per-budget grants, or scoped signatures are absent. In agent-wallet mode, "delegation" reduces to copying the mnemonic (forbidden by T1). In WalletConnect mode, the Trust Wallet mobile app enforces per-session scoping but not per-agent sub-budgets. Fails A2 and A3 delegation needs.

## 6. Relevant protocol coverage

**Chains with full wallet support** (from the bundled `CHAINS` map, `dist/index.js:7220-7260` and surrounding) [S5]: ethereum, arbitrum, optimism, polygon, bsc, avalanche, base, fantom, linea, scroll, zksync, blast, sonic, celo, aurora, solana, bitcoin, dogecoin, litecoin, cosmos, tron, near, aptos, ton, sui.

**Swap namespaces:** `eip155` and `solana` only [S5: `SWAP_SUPPORTED_NAMESPACES = ["eip155", "solana"]`]. Swap routing is aggregated via 1inch, KyberSwap, 0x, Jupiter, THORChain; bridging via Stargate, Synapse, Squid [S3 coverage].

**Stellar support: no.** Stellar (SLIP-44 coin 148) appears in `MAINTAINED_COIN_IDS` in the bundle [S5 `dist/index.js:7227`] and in the REST API's chain table [S1 `skills/api/references/setup.md:271`], meaning the bundled WalletCore WASM can derive a Stellar address and query XLM prices. But there is no Stellar entry in the CLI's `CHAINS` map, no `--chain stellar` option anywhere in the command surface, no Horizon/RPC integration, no Soroban awareness, no SEP coverage. XLM is not sendable, not swappable, not queryable as a wallet balance through TWAK. Roadmap silence on Stellar — no public commitment in any announcement or doc reviewed [S2, S3, S4].

**MCP tool surface** (`twak serve` stdio mode; extracted verbatim from the bundled server, `dist/index.js:45462-46396`) [S5]:
`get_balance`, `get_token_holdings`, `search_assets`, `get_wallet_address`, `connect_wallet`, `get_wallet_status`, `transfer_token`, `swap`, `get_swap_quote`, `get_token_price`, `validate_transaction`, `get_asset_info`, `get_transaction_history`, `get_transaction_details`, `get_swap_history`, `create_alert`, `list_alerts`, `check_alerts`, `delete_alert`, `create_wallet`, `get_address`, `list_addresses`, `wallet_balance`, `token_balance`, `transfer_token`, `approve_token`, `check_allowance`, `get_trending_tokens`, `check_token_risk`, `create_automation`, `list_automations`, `delete_automation`. 32 tools (note `transfer_token` appears twice — one plain-transfer variant and one ERC-20 variant — ambiguity risk for an agent).

**x402:** client and server. `twak x402 request` auto-pays HTTP 402 endpoints; `twak serve --rest --x402` makes the CLI's own REST endpoints payment-gated. Payment assets limited to USDC on Ethereum/Base/Polygon [S1 `x402.md:68-76`].

## 7. Licence and adoption signals

- **Licence:** MIT for both `tw-agent-skills` [S1 `LICENSE`] and the published `@trustwallet/cli` npm package [S5 `package/LICENSE`, `registry.npmjs.org` metadata].
- **Source available:** **partial.** `tw-agent-skills` is a fully public repo (docs only, no CLI source). The CLI monorepo at `github.com/trustwallet/twak` returns 404 — the npm package is the only distributable artifact. The bundled `dist/index.js` (~56k lines, ~2 MB) is a minified tsup build; source not recoverable. WalletCore itself is open source at `trustwallet/wallet-core` (Apache-2.0), consumed here as WASM.
- **Last meaningful commit:** `tw-agent-skills` tip `6f8671f` committed 2026-04-15. `@trustwallet/cli@0.9.0` published 2026-04-15.
- **GitHub stars:** n/a for `twak`; `tw-agent-skills` is <1 month old at research time.
- **Known production users:** Trust Wallet mobile app (~220M installs); integrations advertised with Claude Code, Cursor, Windsurf, Codex, Copilot, Cline, opencode, roo [S1 `README.md:17-23`].
- **Commercial backing:** Trust Wallet (Bitkeep Global Pte Ltd); well-funded incumbent.
- **Telemetry:** Amplitude analytics bundled; off by default, opt-in via `twak telemetry --opt-in` or `TWAK_TELEMETRY` env var [S5 `dist/index.js` — `registerTelemetryCommand`].

## 8. Non-negotiable check

Evaluated separately for each mode because the threat model diverges.

### Mode A — TWAK-native agent wallet

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | partial | Keys on-disk AES-256-GCM + OS keychain; never uploaded. But signing is unattended under agent control — N1's "explicit per-action user consent" clause not met in this mode. |
| N2 | Autonomous (no project-operated backend) | no | Requires HMAC credentials from `portal.trustwallet.com` gateway (`tws.trustwallet.com`), 1 req/s rate limit [S1 `skills/api/references/setup.md:18-19`]. Without Trust Wallet's servers, the CLI cannot resolve tokens, prices, swap quotes, or route swaps. |
| N3 | No central server for keys/policy/history | no | Keys stay local, but all token/asset/price/risk queries flow through `tws.trustwallet.com`; Amplitude analytics on by opt-in but off by default. History on-chain, not TW-held. |
| N4 | Open source, permissive licence | partial | MIT, but CLI source monorepo is private; only minified npm bundle available. Fails the spirit of N4 — no independent audit path. |
| N5 | JSON-default output | no | `--json` is a flag on every command; default output is human-formatted with `chalk`/`boxen` [S5 `dist/index.js:38725`]. Not JSON-default. No `{ ok, data | error, request_id }` envelope — errors surface via non-zero exit code. |
| N6 | Testnet/mainnet parity | no | No testnet switch visible in the CLI; all chain entries are mainnet-only (`chainId: "1"`, etc.) [S5 `CHAINS` map]. Testnet operations are not supported. |

### Mode B — AI agent proposes to user's Trust Wallet via WalletConnect

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | Keys stay in the Trust Wallet mobile app; CLI proposes via `wc_sessionRequest`, user approves in-app [S5 WalletConnect integration]. Approval UI is wallet-owned (§9 / T9). |
| N2 | Autonomous (no project-operated backend) | no | Same gateway dependency as Mode A; additionally WalletConnect relays at `relay.walletconnect.org` are required [S5 `relayUrl: "wss://relay.walletconnect.org"`]. |
| N3 | No central server | no | Gateway + WC relay + (opt-in) Amplitude. |
| N4 | Open source | partial | Same as Mode A. |
| N5 | JSON-default | no | Same as Mode A. |
| N6 | Testnet/mainnet parity | no | Same as Mode A. |

## 9. What to adopt

- **WalletConnect as the delegation-free approval channel** — resistant to T9 because the approval UI lives in a separate wallet-owned process (mobile app), not HTML rendered by the agent. Cite for the Soroban-wallet-signing transport (SEP-43, passkey-kit companion). Answers A4.
- **Per-command USD ceiling with mandatory acknowledgement (`--max-usd`, `--confirm-to`, `--confirm-unlimited`)** [S1 `send.md:66-70`, `erc20.md:34-37`] — simple structural defence against T3. Make these structurally required rather than opt-out flags. Good for A1.
- **`twak chains --json` as the canonical chain discovery surface** — `{ key, name, symbol, namespace, chainId, coinId }` [S1 `setup.md:121-128`] maps cleanly onto an agent's schema needs. Adopt shape for our `stellar-agent-wallet chains`. Answers T10.
- **One binary, three transports (CLI / MCP stdio / REST)** via a single `serve` command with a `--rest` flag — cheap way to cover all agent runtimes without maintaining three repos [S5 `registerServeCommand`]. Answers A1+A2.
- **`x402 request --max-payment` as a per-call cap with interactive confirm by default** — aligns with A3 and T2. Our x402-on-Stellar implementation should copy the cap-and-confirm pattern.
- **SLIP-44 compact asset ID** `c{coinId}[_t{contract}]` [S1 `skills/api/references/setup.md:146-151`] — concise, namespace-safe; useful pattern for Stellar issuer-anchored assets (`xlm` / `usdc.issuer=GA5Z...`). Addresses T3.
- **Opt-in telemetry with `--opt-in` + env var + dedicated `telemetry` command** — good citizen pattern to copy for our CLI. Answers A7.

## 10. What to avoid

- **No local policy engine.** Limits are passed per-invocation as flags the agent constructs; the agent is trusted to pass them. Directly fails T2 and T4. Our design must put policy in the wallet trust boundary (`analysis/02-threat-model.md` §6.1), not in the agent's tool call.
- **No delegation primitive.** No subaccounts, no scoped keys, no revocable capabilities. Fails A2 and A3.
- **Human-formatted default output with JSON behind a flag.** Fails N5 (`analysis/00-context.md` §4 N5). Every command must be JSON-default — do not repeat TWAK's choice.
- **No testnet parity.** Fails N6 and T10. Every chain entry in TWAK is mainnet-only; this is the "wrong-network" failure mode described in A7. Our CLI must require explicit `--network testnet|mainnet` and namespace per-network state.
- **Mandatory gateway (`tws.trustwallet.com`, HMAC-signed, 1 req/s).** Every non-read-only operation depends on Trust Wallet servers for token resolution and swap routing. Fails N2/N3. Even the "self-custodial" claim is conditional on Trust Wallet remaining online and issuing credentials.
- **Duplicate MCP tool names (`transfer_token` registered twice)** [S5 `dist/index.js:45630,46193`] — hallucination amplifier for T3.
- **Closed-source CLI distributed only as a 2 MB minified bundle.** Fails the spirit of N4 and T8; no audit path, no reproducible build, WalletCore WASM loaded via runtime hook (`trust-sdk` polyfill banner [S5 `dist/index.js:1-34`]) — a supply-chain risk surface.
- **Amplitude analytics bundled** [S5] — even off-by-default, shipping a third-party telemetry SDK inside a key-signing binary increases T8 exposure. Ship without.
- **Mnemonic at `~/.twak/wallet.json` on disk** [S1 `setup.md:100-104`] — acceptable for A1 with full-disk encryption assumptions, but our design should prefer TPM/Secure-Enclave-backed storage and ephemeral unlock windows (T1 defence). TWAK's "password mirrored to OS keychain" convenience erodes the AES protection back to the keychain's security boundary.

## 11. Open questions

- Roadmap for Stellar. Unknown — no public commitment; Stellar appears in `MAINTAINED_COIN_IDS` but not in the CLI's CHAINS map, suggesting it's a dormant WalletCore import rather than a committed integration.
- Exact MCP authentication path when `--walletconnect` is used alongside `--rest --x402` — whether the CLI rejects mixing modes.
- Rate-limit behaviour under burst swap usage (1 req/s gateway cap vs. N agents querying).
- Whether `tw-agent-skills` can be reused standalone (without the `twak` CLI) — the skill docs assume `twak` is installed.
- Whether a Stellar contributor could land chain support through `trustwallet/wallet-core` alone without Trust Wallet opening the `twak` source.

## 12. Sources

**S1. Open-source skill repo (local clone).** [`tw-agent-skills/`](https://github.com/trustwallet/tw-agent-skills) @ commit `6f8671f1fa53c9854375d2ae7721a407a23940ff` (committed 2026-04-15). Upstream: https://github.com/trustwallet/tw-agent-skills (accessed 2026-04-18). License: MIT. Canonical source for CLI command surface, REST API shape, chain lists, security model. Primary source.

**S2. Official announcement.** Trust Wallet — "Introducing the Trust Wallet Agent Kit (TWAK)". https://trustwallet.com/blog/announcements/introducing-the-trust-wallet-agent-kit-twak-your-ai-agent-can-now-act-on-crypto (accessed 2026-04-18). Dated 2026-03-26. Source for: two-mode architecture, "CLI and MCP both supported out of the box", chain-count claim, custody posture.

**S3. Follow-up official announcement.** Trust Wallet — "Your AI Agent Can Now Run Your Crypto Strategy — DCA Automation and Limit Orders in TWAK". https://trustwallet.com/blog/announcements/your-ai-agent-can-now-run-your-crypto-strategy-introducing-dca-automation-and-limit-orders-in-trust-wallet-agent-kit (accessed 2026-04-18). Confirms DCA / limit-order delivery on CLI and MCP; confirms mode split for automations.

**S4. Developer portal.** https://portal.trustwallet.com/ (accessed 2026-04-18). Source for install command and env-var names. Does not expose API reference in machine-readable form.

**S5. npm-published CLI bundle.** `@trustwallet/cli@0.9.0` at `https://registry.npmjs.org/@trustwallet/cli/-/cli-0.9.0.tgz` (published 2026-04-15, accessed 2026-04-18, local extraction at `/tmp/twak-cli/package/`). License MIT; repository field points to `github.com/trustwallet/twak` (404 / private). Bundled `dist/index.js` read directly for: MCP tool list (`dist/index.js:45462-46396`), `CHAINS` map (`:7220-7260` + surrounding), `SWAP_SUPPORTED_NAMESPACES`, WalletConnect relay constants, `serve` command options, telemetry plumbing.

**S6. `trustwallet/wallet-core`.** https://github.com/trustwallet/wallet-core (accessed 2026-04-18, not cloned). Apache-2.0. Consumed by TWAK as bundled WASM (`trust-sdk/wallet-core.wasm`). Confirms Stellar derivation support at library level.

## 13. Cross-links

- Compare to `research/external/crypto/kraken-cli.md` — opposite design on N5 (Kraken is JSON-default, TWAK human-default).
- Compare to `research/external/crypto/coinbase-agentic.md` — Coinbase AWAC uses TEE; TWAK uses local HD + OS keychain. Different compromise profile (T1).
- Compare to `research/external/stellar-ecosystem/stellar-mcp-server.md` — MCP-only, Stellar-specific, env-var keys; TWAK is multi-chain, multi-transport, gateway-dependent.
- Feeds requirements on: JSON-default output (N5), local policy engine (T2 defence), delegation primitive (A2/A3), testnet parity (N6/T10), telemetry hygiene (T8).
