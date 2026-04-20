# GitHub CLI (`gh`)

**Status:** complete
**Researcher:** research agent (Opus 4.7)
**Last updated:** 2026-04-18
**ID:** gh-cli
**Category:** non-crypto

---

## 1. What it is

`gh` is GitHub's official command-line client for repositories, issues, pull requests, Actions workflows, releases, Codespaces, and the REST/GraphQL APIs. It is a Go binary distributed for macOS, Linux, and Windows; ~60k stars on `github.com/cli/cli`; MIT-licensed. It is the incumbent developer tool for scripting GitHub from a shell and is increasingly used as a tool invocation target by coding agents [1][2].

## 2. Agent interface

CLI only. The CLI is also an official SDK wrapper via the `github.com/cli/go-gh/v2` Go library, which exposes `gh.Exec(args...)` as a subprocess-capture API — the sanctioned way for third-party programs to drive `gh` [3]. A programmatic caller targeting GitHub directly typically runs `gh auth login --web` once, then issues commands like `gh pr list --json number,title,author --jq '.[] | select(.author.login=="me")'` [1]. No RPC surface is exposed; the CLI is the interface.

## 3. Custody model

`gh` holds OAuth bearer tokens on behalf of the human. On successful login, the token is written to the OS credential store via the `zalando/go-keyring` library — `security` CLI on macOS Keychain, DPAPI on Windows Credential Manager, and the Secret Service D-Bus interface (GNOME Keyring / KWallet) on Linux. If no credential store is reachable, `gh` falls back to a plain-text file under `$GH_CONFIG_DIR` (default `$XDG_CONFIG_HOME/gh`), and `--insecure-storage` forces that fallback [4][5][6]. There is no concept of "user signs, CLI relays" — the token is a bearer credential and anything on the host with read access to the keyring entry can act as the user.

## 4. Policy and approval model

None in the client. Scopes are negotiated at login time (`repo`, `read:org`, `gist` by default) and modified via `gh auth refresh --scopes ...`, which only re-runs the OAuth flow to broaden or reset scopes — it is not a token-refresh mechanism [7]. All enforcement happens server-side at GitHub. `gh` has no local allowlist, amount cap, or per-command confirmation gate.

## 5. Delegation model

None in-band. Delegation is handled out of band by issuing a fine-grained Personal Access Token and passing it through the `GH_TOKEN` / `GITHUB_TOKEN` / `GH_ENTERPRISE_TOKEN` environment variables, which take precedence over the stored credential [6]. This is how CI runners authenticate — a short-lived token is injected as a secret.

## 6. Relevant protocol coverage

- **Auth scheme:** OAuth web flow (default) with device-code-style browser handshake; PAT via stdin or env var [4].
- **Output:** `--json <comma-separated-field-list>` produces a JSON array/object; `--jq <expr>` applies a jq filter to that output with implicit `--raw-output`; `--template <go-template>` applies a Go template with helpers like `tablerow`, `timeago`, `truncate` [1][8]. `--json` without arguments prints the available field list and exits — a self-describing discovery mechanism.
- **Pagination:** opt-in via `--paginate` on `gh api`; `--slurp` wraps multi-page results into a single array. Pagination is exposed, not hidden [9].
- **Rate limits:** not abstracted. A 403 surfaces as an HTTP error and exits non-zero; callers are expected to poll `gh api rate_limit` themselves [10].
- **Errors:** `gh` does not wrap errors in a JSON envelope even when `--json` is set. Error text goes to stderr; stdout is either the successful JSON payload or empty [11]. Exit codes: `0` success, `1` generic failure, `2` cancelled, `4` auth required [12].
- **Default context:** `gh repo set-default` pins an `owner/repo`; `GH_REPO=[host/]owner/repo` overrides; `-R owner/repo` overrides per-command; otherwise `gh` infers from the Git remote [13][6].

## 7. Licence and adoption signals

- Licence: MIT.
- Source available: yes (`github.com/cli/cli`).
- Stars: ~60k.
- Distribution: shipped by Homebrew, winget, apt, GitHub Actions runners.
- Ecosystem: `go-gh` library for programmatic wrapping [3]; extensions directory at `github.com/topics/gh-extension`.

## 8. Applicability to an agent-wallet CLI

Non-crypto reference only: `gh`'s custody posture (long-lived bearer token in keyring, no per-action user gate, no policy locus) is the *opposite* of what a Stellar AI-agent wallet must provide. Its value is as a CLI-UX and agent-affordance reference — specifically `--json` field selection, `--jq` filtering, self-describing field lists, and extension mechanics — not as a trust or custody template.

## 9. What to adopt

- **`--json <fields>` + `--jq <expr>` + `--template <go-tmpl>` triad.** Field-list selection keeps payloads small; jq filtering means callers do not parse; templates cover human output. This is the incumbent pattern agents already handle. Relevant to A1, A2 (all need deterministic scriptable output; N5).
- **Self-describing field discovery.** `--json` with no value prints the valid field list and exits 0. An agent can enumerate a command's output schema without out-of-band docs. Relevant to A7.
- **`GH_CONFIG_DIR` + XDG-respecting config location with env override.** Clean separation of binary, config, and credential stores; trivially reproducible on ephemeral CI hosts (A5).
- **OS-keyring-first credential storage with explicit `--insecure-storage` opt-out.** The pattern — not the token model — is the lesson: default to the strongest local store, require a named flag to degrade. Relevant to A1.
- **Standardised exit codes including a dedicated "auth required" code (`4`).** Lets a caller distinguish "broken setup" from "command failed on merits" without parsing text [12]. Relevant to A1, A5.
- **Env-var precedence for automation (`GH_TOKEN` > keyring).** Lets CI/CD inject credentials without touching the on-disk store (A5).
- **Default-context command (`gh repo set-default`) with `-R` per-invocation override and `GH_REPO` env override.** Translate to `stellar-wallet account set-default` / `--account` / `STELLAR_ACCOUNT` for A7 ergonomics without losing per-command explicitness.

## 10. What to avoid

- **Long-lived bearer token with broad scopes stored at rest.** Directly violates our threat model: host compromise is total account compromise with no blast-radius bound (A1, A2). Our design must sign per-transaction, not hold a static bearer.
- **No local policy layer at all.** Agents that run unattended (A1, A3) cannot rely on server-side enforcement — there is no Stellar-side equivalent of GitHub's server-side scope check for arbitrary SEP or Soroban flows. Policy must be local (see `01-actor-model.md` §5 item 3).
- **Errors not wrapped in a JSON envelope when `--json` is set.** `gh` leaves the caller to branch on exit-code + stderr parsing [11]. We should instead ship `{ok: false, error: {code, message, request_id}, data: null}` so a single parse path serves both outcomes.
- **Rate-limit handling left to the caller.** Fine for `gh`'s audience but a poor default for an agent that might retry blindly; our network layer should at minimum expose a typed `RATE_LIMITED` error with a `retry_after_ms`.
- **`gh extension install owner/gh-foo` fetches and runs arbitrary binaries with zero signature verification or sandboxing.** An extension inherits the user's `gh` token and the user's shell. For a wallet, this is a supply-chain cliff — extensions must be signed and scope-capability-bounded.
- **`--json` coverage is not universal.** Some commands (auth status, many mutations) do not support `--json`; `gh auth status` even writes success output to stderr [11][14]. Our CLI should make JSON output a universal invariant, not a per-command opt-in.
- **`gh auth refresh` is misleadingly named.** It re-prompts for scopes, not a token-lifecycle refresh [7]. Name local commands by what they do (`scope-grant`, `scope-revoke`) rather than by what they resemble in other systems.

## 11. Open questions

- Precise behaviour when a refresh token expires or is revoked server-side — is the failure distinguishable from a scope error at the CLI layer, or does the user see the same generic auth prompt?
- Whether the extension mechanism ever plans signature verification — relevant only because we will be asked "why not copy `gh extension`?".

## 12. Sources

1. GitHub CLI manual — formatting flags, accessed 2026-04-18, https://cli.github.com/manual/gh_help_formatting — primary reference for `--json` / `--jq` / `--template` syntax.
2. GitHub CLI top-level manual, accessed 2026-04-18, https://cli.github.com/manual/ — command taxonomy and environment variables.
3. `cli/go-gh` package docs, accessed 2026-04-18, https://pkg.go.dev/github.com/cli/go-gh/v2 — `gh.Exec()` subprocess wrapper API.
4. `gh auth login` manual, accessed 2026-04-18, https://cli.github.com/manual/gh_auth_login — device/web login, credential store, `--insecure-storage`.
5. `cli/cli` discussion #8980 on OAuth token keychain usage, accessed 2026-04-18, https://github.com/cli/cli/discussions/8980 — confirms `zalando/go-keyring` via D-Bus Secret Service on Linux.
6. GitHub CLI environment variables reference, accessed 2026-04-18, https://cli.github.com/manual/gh_help_environment — `GH_TOKEN`, `GH_HOST`, `GH_REPO`, `GH_CONFIG_DIR` precedence.
7. `gh auth refresh` manual, accessed 2026-04-18, https://cli.github.com/manual/gh_auth_refresh — scope-management semantics, not token lifecycle.
8. `gh issue list` manual, accessed 2026-04-18, https://cli.github.com/manual/gh_issue_list — enumerated `--json` field set.
9. `gh api` manual, accessed 2026-04-18, https://cli.github.com/manual/gh_api — `--paginate`, `--slurp`, GraphQL cursor contract.
10. `cli/cli` issue #10426 on search rate-limit surfacing, accessed 2026-04-18, https://github.com/cli/cli/issues/10426 — confirms rate-limit is surfaced as raw HTTP 403.
11. `cli/cli` issue #7447 on `gh auth status` writing to stderr on success, accessed 2026-04-18, https://github.com/cli/cli/issues/7447 — stream-uniformity gap.
12. `gh help exit-codes`, accessed 2026-04-18, https://cli.github.com/manual/gh_help_exit-codes — `0`/`1`/`2`/`4`.
13. `gh repo set-default` manual, accessed 2026-04-18, https://cli.github.com/manual/gh_repo_set-default — default-context management.
14. `gh extension install` manual, accessed 2026-04-18, https://cli.github.com/manual/gh_extension_install — binary vs. script extensions, no signature verification documented.

## 13. Cross-links

- CLI-UX patterns: compare to `stellar-cli` (command taxonomy), `aws-cli` (profile + env precedence), `stripe-cli` (JSON envelope shape).
- Reinforces `01-actor-model.md` §5 item 3's observation that policy enforcement must be local for unattended actors (A1, A2, A3) — `gh`'s server-enforced-only model cannot be copied.
- Feeds candidate requirements: universal JSON output with error envelope; `--json` field discovery; OS-keyring-by-default storage with named opt-out; standardised exit codes including auth-required.
