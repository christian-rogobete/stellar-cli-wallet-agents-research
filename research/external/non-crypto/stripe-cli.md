# Stripe CLI

**Status:** complete
**Researcher:** research agent (Opus 4.7)
**Last updated:** 2026-04-18
**ID:** stripe-cli
**Category:** non-crypto

---

## 1. What it is

`stripe-cli` is Stripe's official developer tool, written in Go and distributed as a single binary, for driving the Stripe payment API from a terminal. It supports resource CRUD against the REST API, local webhook forwarding, event triggering, log tailing, fixtures, and a plugin mechanism [1][2]. The closest non-crypto analogue to a financial-API agent CLI: the object model is resources (customers, charges, payment intents), the network split is test vs. live, and idempotency is first-class on the wire protocol [3][4].

## 2. Agent interface

CLI over HTTPS to `api.stripe.com`. A programmatic caller sequences: `stripe login` (persists a restricted key at `~/.config/stripe/config.toml` [5]) → resource or HTTP command. Resource form: `stripe customers create --name "Jane"`; generic form: `stripe post /v1/customers -d name=Jane -i foobar123456` [4]. All POST mutations accept `-i, --idempotency` and `-e, --expand`; GETs accept `--limit`, `--starting-after`, `--ending-before` [4]. Output is JSON on stdout, and `stripe get /subscriptions -d status=past_due | jq '.data[].id' | xargs stripe delete /subscriptions/%` is the canonical composition pattern [4]. For event ingestion the caller runs `stripe listen --forward-to localhost:4242/webhook`, which prints a per-session webhook signing secret and streams events over a WebSocket from Stripe's side [6].

## 3. Custody model

Out of scope for Stripe — the "key" is a bearer API secret, not a signing key. `stripe login` provisions a restricted key (test + live) scoped to "Stripe CLI permissions" and valid 90 days [5]. Storage: `~/.config/stripe/config.toml` by default; on macOS live keys move to Keychain (issue #933 confirms the OS-keyring branch is active for live mode) [7]. Per-command override: `--api-key` bypasses the config entirely [5]. There is no hardware-signing path and no per-transaction approval gate — stolen keys act instantly.

## 4. Policy and approval model

Policy lives server-side in Stripe's dashboard (restricted-key scopes) and is configured out-of-band via the web UI, not the CLI [5]. Client-side the CLI has no policy engine, no spend caps, no destination allowlist. The only client-enforced confirmation is `delete`'s `-c, --confirm` prompt [4]. Live/test separation is implicit from key type: a test key silently becomes a no-op against live resources (`"You cannot use the test ID … in livemode"` returned by the server) [8].

## 5. Delegation model

None in the CLI. Delegation is expressed by creating a differently-scoped restricted key in the dashboard and passing it via `--api-key` or a named profile (`--project-name`, short `-p`) [9]. Revocation = rotate the key in the dashboard. There is no Stripe-native capability model below the API-key boundary.

## 6. Relevant protocol coverage

- **Auth:** bearer restricted key, 90-day TTL, separate test/live values per profile [5].
- **Idempotency:** `Idempotency-Key` header set via `-i` flag; semantics: "saving the resulting status code and body of the first request … regardless of whether it succeeds or fails," keys prunable after 24h, parameter-mismatch on reuse errors [3][4].
- **Pagination:** cursor-based (`starting_after`, `ending_before`) exposed as flags [4].
- **Events / webhooks:** `stripe listen` opens a WebSocket to Stripe and POSTs each event to `--forward-to`; per-session signing secret printed to stderr [6]. `stripe trigger <event>` fabricates fixture events in test mode.
- **Plugins:** `stripe plugin install|upgrade` under `pkg/cmd/plugin.go` / `pkg/cmd/plugins/*.go`; Stripe Apps ships as the `apps` plugin [10][2].

## 7. Licence and adoption signals

- Licence: Apache-2.0 [1]
- Source available: yes (`github.com/stripe/stripe-cli`, Go 98.1 %) [1]
- Last meaningful release: v1.40.6, 2026-04-15 [1]
- GitHub stars: ~2,000 [1]
- Known production users: de-facto tool for Stripe integrators; shipped as VS Code extension.
- Commercial backing: Stripe, Inc.
- Ecosystem traction: dockerised (`stripe/stripe-cli`), Homebrew, Scoop, Linux packages [1].

## 8. Applicability to an agent-wallet CLI

Non-crypto — the non-negotiable check does not apply. Stripe CLI is useful as a **shape reference only**: its resource-oriented command taxonomy, idempotency-key surfacing, `stripe listen` WebSocket pattern, profile-per-account configuration, and HTTP-verb escape hatches (`get`/`post`/`delete`) map cleanly onto a Stellar wallet CLI. Its custody and policy models do not map at all — a bearer API key with server-side scope is the opposite of self-custody (N1) and central (fails N3). Adopt the ergonomics; discard the trust model.

## 9. What to adopt

- **Resource-noun + verb command shape.** `stripe customers create`, `stripe charges retrieve`, backed by a generic `stripe {get,post,delete} /v1/<path>` escape hatch [4]. Maps directly to `stellar accounts create` / `stellar payments send` / `stellar soroban invoke` with a raw `stellar tx submit` underneath. Addresses A5.
- **First-class `--idempotency` flag on every mutation**, documented as "prevents replaying the same requests within a 24 hour period" [4]. For Stellar this maps to deterministic client-side request IDs for the submission layer (retry-safe submission around sequence-number collisions). Addresses A1, A5.
- **Named profiles via `--project-name`/`-p`** with a single TOML config (`~/.config/stripe/config.toml`) [5][9]. We need the same for multiple mainnet / testnet / per-agent key scopes.
- **`stripe listen` pattern:** a subprocess that opens a persistent stream and forwards events to a local HTTP endpoint, printing a signing secret for the session [6]. Direct analogue for Horizon SSE / Soroban RPC `getEvents` subscription, wrapped as `stellar listen --forward-to …`. Addresses A1, A3, A6.
- **`stripe trigger` for fixture events** [1]. Our equivalent: deterministic testnet scenario generators (`stellar trigger payment-received`) for agent integration tests. Addresses A7.
- **`stripe login` flow that provisions scoped credentials rather than handling raw secrets** [5]. We cannot do this server-side, but the pattern of "CLI bootstraps its own narrowest-possible credential" is the right UX: on Stellar, derive a scoped signer or smart-account policy entry, not the root key.

## 10. What to avoid

- **Implicit test/live selection via which key happens to be loaded.** `--live` is sparsely documented, users regularly end up in the wrong mode [8], and the safety rail is a server-side error rather than a structural check. Our T10 requires explicit `--network` on every command with no ambient default.
- **Bearer credential as the only auth primitive.** A stolen `rk_live_…` is instant game-over, with no per-action approval, no spend cap, no hardware gate. Fails N1 and the T1/T2 defences.
- **Policy in the vendor dashboard.** Restricted-key scopes are edited in a web UI the agent cannot see or enforce [5]. Fails N2/N3 and leaves T4 with no local enforcement point.
- **No JSON envelope; raw API JSON on stdout.** Convenient for `jq` chaining but provides no `ok/error/request_id` wrapper for agent error-handling. Addresses A1 (tooling consistency) and `kraken-cli` comparison.
- **`--skip-verify` for TLS on `listen`** [11]. Acceptable for a test-only tool; unacceptable for a signing tool. Rules out as a pattern for mainnet operations under T7.
- **Plugins without a hardening story.** `stripe plugin install` exists but the architecture doc does not describe sandboxing or signature verification of plugin binaries [2]. T8 demands we do not copy this shape without adding plugin isolation.

## 11. Open questions

- Does `stripe listen` resume event delivery after reconnect, and with what guarantees? (Relevant if we mimic the pattern for Horizon SSE, which is at-least-once.)
- How does the CLI handle key rotation mid-session when the dashboard revokes the restricted key? (Graceful 401 with relogin prompt, or crash?)
- Is the plugin manifest signed, and against what root? `ARCHITECTURE.md` does not say [2].

## 12. Sources

1. `github.com/stripe/stripe-cli` — README, releases, licence. Accessed 2026-04-18.
2. `github.com/stripe/stripe-cli/blob/master/ARCHITECTURE.md` — command system, request flow, plugin layout. Accessed 2026-04-18.
3. `docs.stripe.com/api/idempotent_requests` — Idempotency-Key semantics, 24h TTL, parameter-mismatch behaviour. Accessed 2026-04-18.
4. `github.com/stripe/stripe-cli/wiki/http-(get,-post-&-delete)-commands` — `-i/--idempotency`, `-d/--data`, `-e/--expand`, pagination flags, `jq` composition example. Accessed 2026-04-18.
5. `docs.stripe.com/stripe-cli/keys` — `stripe login` behaviour, restricted keys, 90-day TTL, `~/.config/stripe/config.toml`, `--api-key` override. Accessed 2026-04-18.
6. `docs.stripe.com/webhooks/test` — `stripe listen --forward-to`, per-session signing secret. Accessed 2026-04-18.
7. `github.com/stripe/stripe-cli/issues/933` — macOS Keychain storage of live-mode keys. Accessed 2026-04-18.
8. `stackoverflow.com/questions/78828633` and `github.com/stripe/stripe-cli/issues/288`, `/issues/1261` — `--live` flag behaviour, documentation gaps, server-side "test ID in livemode" error. Accessed 2026-04-18.
9. `docs.stripe.com/cli/flags` — global flags: `--api-key`, `--project-name/-p`, `--config`, `--device-name`, `--log-level`, `--color`. Accessed 2026-04-18.
10. `docs.stripe.com/stripe-apps/reference/cli` — `stripe plugin upgrade apps`; plugin as the Stripe Apps delivery mechanism. Accessed 2026-04-18.
11. `docs.stripe.com/stripe-cli/use-cli` — `--skip-verify`, `--load-from-webhooks-api`, resource-command examples. Accessed 2026-04-18.

## 13. Cross-links

- Compare to `research/external/crypto/kraken-cli.md` — JSON envelope, MCP server pattern, idempotency surfacing.
- Compare to `research/external/non-crypto/gh-cli.md` and `aws-cli.md` — profile management, `--profile` vs `--project-name` convergence.
- Compare to `research/external/crypto-stellar/stellar-cli.md` — current Stellar CLI command taxonomy and network-selection mechanism.
- Feeds requirements around: resource-noun command shape (R-cli-shape), mandatory `--network` (R-net-scoping, T10), `--idempotency`-equivalent submission identifiers (R-submit-idempotent), local event-stream subprocess (R-listen).
