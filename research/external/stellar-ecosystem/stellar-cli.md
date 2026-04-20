# stellar-cli (official Stellar CLI)

**Status:** complete
**Researcher:** research-agent
**Last updated:** 2026-04-18
**ID:** stellar-cli
**Category:** crypto-stellar

---

## 1. What it is

SDF's official CLI for Stellar classic and Soroban. Rust workspace, two binaries (`stellar`, alias `soroban`), plus public library crate `soroban_cli` (Apache-2.0, v26.0.0). Covers identities, networks, classic ops, Soroban lifecycle, XDR/strkey, events, ledger/snapshot, external-binary plugins. Developer-facing; assumes human-readable output (`cmd/soroban-cli/src/commands/mod.rs:32-68 @ 8097cf5`).

## 2. Agent interface

CLI only. No MCP, no RPC server, no daemon. Caller shells out and parses stdout. Minimum signed payment:

```
stellar keys generate alice --secure-store
stellar keys fund alice --network testnet
stellar tx new payment --source alice --destination G... \
    --amount 10000000 --asset native --network testnet
```

`tx new` builds/signs/sends by default; `--build-only` emits XDR that pipes through `tx sign` and `tx send`. Only `tx send` emits JSON unconditionally (`cmd/soroban-cli/src/commands/tx/send.rs:57 @ 8097cf5`).

## 3. Custody model

Self-custodial, host-local. Per-identity TOML under `$XDG_CONFIG_HOME/stellar/identity/<name>.toml` with `0o700` enforced on unix (`cmd/soroban-cli/src/config/locator.rs:113-171,496-548 @ 8097cf5`). Four signer backends (`cmd/soroban-cli/src/config/secret.rs:57-64 @ 8097cf5`): `SecretKey` (plaintext `S...`), `SeedPhrase` (plaintext SEP-5 phrase, **default from `keys generate`**), `SecureStore` (OS keyring via `--secure-store`, behind the `additional-libs` feature — `signer/secure_store.rs:29-66 @ 8097cf5`), `Ledger` hardware wallet (`signer/ledger.rs:21-60 @ 8097cf5`). Opt-in `--sign-with-lab` delegates signing to lab.stellar.org.

## 4. Policy and approval model

None. No spend cap, counterparty allowlist, rate limit, time window, destination check. Grep across the sources for policy/limit/allowlist returns only per-op fields (e.g. `max_amount_a` on LP ops). Signing depends only on "identity exists and unlockable"; unattended use has no human-approval gate. Network is required per call (`cmd/soroban-cli/src/config/network.rs:20-80 @ 8097cf5`), but `network use` / `STELLAR_NETWORK` persists a default that can silently apply.

## 5. Delegation model

None at the CLI layer. Delegation = share a key file or seed phrase; `--hd-path` is derivation, not a trust boundary. Soroban `SorobanAuthorizationEntry` signing works for ed25519; contract-address credentials (smart accounts, SEP-45) are explicitly rejected ("This address is for a contract… Currently the CLI doesn't support that yet" — `cmd/soroban-cli/src/signer/mod.rs:113-120 @ 8097cf5`).

## 6. Relevant protocol coverage

Top-level commands (`commands/mod.rs:142-222 @ 8097cf5`): `contract`, `keys`, `network`, `tx`, `ledger`, `events`, `fees`, `snapshot`, `container`, `message`, `xdr`, `strkey`, `cache`, `config`, `env`, `plugin`, `completion`, `doctor`, `version`. All 22 classic ops under `commands/tx/new/`. Soroban: `build|upload|deploy|invoke|extend|restore|read|fetch|bindings|alias`. SEP-5 HD derivation; SEP-53 `message sign|verify`. No SEP-10, SEP-24, SEP-45, or x402 surface.

## 7. Licence and adoption signals

Apache-2.0; source at `github.com/stellar/stellar-cli`; workspace v26.0.0 (`Cargo.toml:24 @ 8097cf5`); HEAD 2026-04-15. SDF-backed. Reference CLI for every Soroban tutorial; Homebrew, `cargo install`, nix flake, GitHub Action. Stars / MAU not captured (shallow clone).

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | Host-local keys only |
| N2 | Autonomous | yes | No SDF backend at runtime; `--sign-with-lab` opt-in |
| N3 | No central server | yes | State in `$XDG_CONFIG_HOME/stellar`; only opt-out upgrade check |
| N4 | Open source, permissive | yes | Apache-2.0 |
| N5 | JSON-default output | **no** | Only `tx send|fetch`, `ledger fetch|latest`, `events`, `network info|health|settings`, `fees stats`, `tx decode|encode` expose `--output json`. `keys ls|generate`, `contract invoke`, all `tx new *`, `tx sign`, `contract deploy`, `network ls`, `plugin ls` print free-form text (grep for `OutputFormat`: ~9 wire it) |
| N6 | Testnet/mainnet parity | yes | Same commands; network required per call (`config/network.rs:20-80 @ 8097cf5`) |

## 9. What to adopt

- **Library crate as embed surface.** `soroban_cli` exposes `commands::Root`, `config::*`, `signer::*`, `tx::*`, `assembled::*`, `resources::*` (`cmd/soroban-cli/src/lib.rs:14-30 @ 8097cf5`). Reuse SEP-5 derivation, secret format, ledger signer, Soroban tx assembly without forking (A2, A5).
- **Identity file layout.** TOML-per-identity, `0o700` tree (`locator.rs:528-548 @ 8097cf5`). Same layout gives the human (A7) a recovery path in plain `stellar`.
- **OS-keyring integration.** `secure_store::sign_tx_data` is a clean signing boundary (`secure_store.rs:58-65 @ 8097cf5`); adopt the `secure_store:` entry-prefix convention for cross-tool interop (T1).
- **Plugin mechanism.** Any `stellar-<name>` on `PATH` becomes `stellar <name>` (`commands/plugin/default.rs @ 8097cf5`). Ship our wallet as `stellar-agent-wallet` and inherit discovery (option-A lever).
- **SEP-53 `message sign|verify`** reusable for SEP-10 challenge signing (A3).
- **`Signer`/`SignerKind` enum** (`signer/mod.rs:215-226 @ 8097cf5`) is the right extension point for passkey/smart-account backends.

## 10. What to avoid

- **Unenforced policy (T2, T4, T6).** Unbounded signing blocks A1/A2/A5.
- **Plaintext seed phrase default (T1).** `keys generate` writes the phrase to TOML unless `--secure-store` is passed (`commands/keys/generate.rs:115-126 @ 8097cf5`). Invert the default.
- **Inconsistent JSON (N5, T3).** Agents scraping `keys ls` / `contract invoke` text are one format tweak away from T3. Our wrapper needs a uniform envelope on every exposed command.
- **No smart-account signing.** `ScAddress::Contract` credentials rejected (`signer/mod.rs:113-120 @ 8097cf5`); A2/SEP-45 must be added on top.
- **`--sign-with-lab`** ships the tx hash to an SDF web service — block outright (N2).
- **No MCP or structured IPC.** Wrapping requires a separate MCP/JSON-RPC layer; nothing in-process to reuse.
- **Ambient network default (T10).** `STELLAR_NETWORK` / `network use` persist and can silently flip networks. Require network named per call.

**Key decision input (§9/§10):** we can build on top. `soroban_cli` is a real library crate with a `pub` module surface, and the plugin mechanism gives first-class integration as `stellar-agent-wallet`. Ship alongside only what the CLI is missing — policy engine, C-account signer, uniform JSON envelope, MCP server, approval channel — and import the rest.

## 11. Open questions

- Is `soroban_cli` usable as a crates.io library dep, or only as a workspace dep (`Cargo.toml:27-30`)? External consumption untested here.
- Stability of `pub commands::*` across 26.x — undocumented.
- `--secure-store` availability in binstall/brew channels (`additional-libs`) — unverified; `--no-default-features` disables it.
- GitHub stars / install telemetry — shallow clone, not captured.

## 12. Sources

Local clone at HEAD `8097cf5158323a91472e32156d3ad21e6a560373` (2026-04-15). All paths under [`stellar-cli/`](https://github.com/stellar/stellar-cli) @ 8097cf5:

- `README.md`, `LICENSE`, `Cargo.toml`, `FULL_HELP_DOCS.md`
- `cmd/soroban-cli/Cargo.toml` (`additional-libs` feature); `cmd/soroban-cli/src/lib.rs` (pub surface)
- `cmd/soroban-cli/src/commands/mod.rs` (top-level tree); `{keys,contract,tx,network,plugin,message}/mod.rs`
- `cmd/soroban-cli/src/commands/{keys/generate.rs, tx/send.rs, plugin/default.rs, global.rs}`
- `cmd/soroban-cli/src/commands/tx/new/` (22 op files)
- `cmd/soroban-cli/src/config/{locator,secret,sign_with,network}.rs`
- `cmd/soroban-cli/src/signer/{mod,secure_store,ledger}.rs`

Secondary: Brave web search 2026-04-18 "stellar-cli MCP server 2026 agent" — confirms no first-party MCP; x402-MCP is a separate SDF effort (`stellar.org/blog/foundation-news/x402-on-stellar`). Upstream: `https://github.com/stellar/stellar-cli` (tip `8097cf5`).

## 13. Cross-links

- Feeds `analysis/06-option-a-cli-first.md`: option-A scope = `stellar-cli` + the pieces §10 lists.
- Contrasts `research/external/crypto/kraken-cli.md` (uniform `--json`, MCP, error envelope).
- Compares to `research/external/stellar-ecosystem/stellar-mcp.md` (third-party MCP) and `freighter.md` (approval channel for T9).
- Surfaces requirements for `analysis/03-requirements.md`: JSON envelope (N5), policy engine (T2/T4/T6), smart-account signer (A2/SEP-45), network-explicit-per-call (T10).
