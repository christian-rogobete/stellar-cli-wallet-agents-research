# AWS CLI

**Status:** complete
**Researcher:** research agent
**Last updated:** 2026-04-18
**ID:** aws-cli
**Category:** non-crypto

---

## 1. What it is

Amazon's first-party CLI for driving every AWS service API. A thin dispatcher over ~400 service clients sharing one configuration system, one credentials chain, one output formatter, and one client-side filter (JMESPath). For this project the interest is configuration and multi-identity posture, not the service surface.

## 2. Agent interface

CLI only; no official MCP. Shape: `aws <service> <op> [--profile P] [--region R] [--output FMT] [--query EXPR]`. Minimum signed-action sequence: (1) place credentials (`[profile P]` in `~/.aws/config` plus `[P]` in `~/.aws/credentials`, or export `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`); (2) `aws sts get-caller-identity --profile P --output json`; (3) run the operation. SigV4 signing is transparent [1][2].

## 3. Custody model

Credentials live in `~/.aws/credentials` (plaintext), environment, an IMDS instance role, a container task role, the IAM Identity Center token cache (`~/.aws/sso/cache`), or an external `credential_process` helper emitting JSON on stdout [3][4]. Signing is local; "who can sign" reduces to "who can read the credential source." No host-bound key protection or hardware-signer integration in the base tool; no approval step between process and API call.

## 4. Policy and approval model

No client-side policy. All authorization is server-side IAM. `credential_process` is the one hook where an operator can inject a local gate (YubiKey, MFA) by refusing to return credentials [4]. Revocation is IAM-side.

## 5. Delegation model

IAM role assumption. A profile with `role_arn` + `source_profile` causes the CLI to call `sts:AssumeRole`, caching short-term credentials under `~/.aws/cli/cache` and re-assuming on expiry [2][3]. `AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN` drive OIDC assume-role, the canonical CI pattern [5]. Revocation takes effect at next token refresh.

## 6. Relevant protocol coverage

SigV4/SigV4a on AWS HTTPS endpoints. Output: bare JSON/YAML/text/table, no `{ok, data, error}` envelope; errors surface as non-zero exit and a free-form stderr message. Pagination is service-specific (`NextToken`, `Marker`, `--starting-token`). No uniform idempotency key. No webhook model.

## 7. Licence and adoption signals

- Licence: Apache-2.0 (`aws/aws-cli`).
- Source available: yes. Actively maintained (v2 ongoing). ~16k GitHub stars.
- Known production users: every AWS-using organisation. Commercial backing: AWS.
- Ecosystem traction: the `~/.aws/config`+`credentials` format and JMESPath `--query` idiom are copied by every major cloud CLI.

## 8. Applicability to an agent-wallet CLI

Non-negotiable check does not apply (non-crypto). AWS CLI predates the agent era but solved the multi-identity ergonomics problem at a scale no crypto CLI has reached. Worth borrowing: the config/credentials split, named profiles, a documented precedence chain, the `credential_process` extension point, and client-side output filtering. Consciously reject: the missing uniform output envelope, per-service `--output text` drift, and policy-entirely-server-side — none tolerable in a self-custodial wallet with no "IAM" fallback.

## 9. What to adopt

- **Two-layer config with one name across both layers.** `~/.aws/config` (region, output, profile graph) is separated from `~/.aws/credentials` (secrets), both keyed by the same profile name [1]. Mirror as non-secret TOML plus OS-keyring secret store addressed by the same identity. Addresses A1/A2/A5.
- **Documented single-source precedence chain.** AWS publishes one canonical order: command-line → env → assume-role → web-identity → SSO → credentials file → config file → custom process → container → IMDS [2]. Makes "why did it pick this key" answerable. Ship the same from day one.
- **`AWS_PROFILE` as the selector env var.** One env var selects identity, overriding file default, overridden by `--profile` [6]. Lets A5/A1/A7 pick identity without editing config. Mirror as `STELLAR_WALLET_PROFILE`.
- **`credential_process` as the external-signer hook.** A config line names an executable returning JSON-with-expiry on stdout [4]. Same shape as a hardware-wallet or passkey-signer bridge; expose signer plugins this way rather than baking them in.
- **One query language over structured output.** `--output json` plus `--query '<JMESPath>'` gives declarative extraction without shelling to `jq` [7]. `gh` uses `--jq` for the same purpose — the language is secondary; *one* in-tool language is primary. `--output off` (stdout suppressed, exit code honoured) is the right primitive for dry-run and policy-check paths [8].

## 10. What to avoid

- **No uniform output envelope.** Every service returns its own top-level shape; no `{ok, data | error, request_id}` wrapper, so one error path cannot serve all commands. Documented workaround is "use `--query` to project into your own shape" [7]. Every wallet command must emit the same envelope. Covers A1/A3/A5.
- **`--output text` is structurally unsafe.** Docs: "We strongly recommend that if you specify `text` output, you also always use the `--query` option" because columns are alphabetised by JSON key and differ across resources and versions [7]. Issue #2418: `describe-stack-events` returns multiple rows where `describe-volumes` returns one for the same flag shape [9]. Derive human views deterministically from the JSON tree; never ship a mode that drifts per service or silently reorders on schema change.
- **`[profile name]` vs `[name]` prefix split between files.** "When naming the profile in a `config` file, include the prefix word `profile`, but do not include it in the `credentials` file" [1]. One naming rule across all wallet state files.
- **Policy locus entirely off-host.** AWS has no client-side policy; every decision is IAM. Our N2/N3 forbids a project-run server — local policy cannot be optional. Covers A1/A2.
- **Cross-service flag and idempotency drift.** Services variously use `--filter`, `--filters`, `--filter-expression` for the same concept [7]; some accept a client request token, most do not, and the CLI synthesises none — a footgun for retrying agents. Enforce a single flag taxonomy and a default idempotency key on every state-changing command (A5).

## 11. Open questions

- What does a signer-plugin contract analogous to `credential_process` look like when the "credential" is a signature over a specific transaction envelope rather than a bearer token?
- Does `AWS_PROFILE` semantics plus a matching `--profile` generalise cleanly to a wallet where the selected identity also determines network, policy file, and RPC endpoint — or do those need separate selectors?
- How much of the cross-service drift is inherent to AWS's scope versus avoidable with strict generator discipline? The answer constrains how far a single-binary wallet can go before the same entropy appears.

## 12. Sources

All accessed 2026-04-18. Primary AWS documentation only.

1. `Configuration and credential file settings in the AWS CLI` — https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html — file format, `[profile name]` vs `[name]` split, named profiles [1].
2. `Authentication and access credentials for the AWS CLI` — https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-authentication.html — canonical precedence order (command-line → env → assume-role → web-identity → SSO → credentials → config → custom process → container → IMDS) [2].
3. `AWS SDKs and Tools standardized credential providers` — https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html — provider chain semantics, caching, refresh [3].
4. `Sourcing credentials with an external process in the AWS CLI` — https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sourcing-external.html — `credential_process` contract, JSON-on-stdout format, expiry handling [4].
5. `Configuring environment variables for the AWS CLI` — https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html — `AWS_WEB_IDENTITY_TOKEN_FILE`, `AWS_ROLE_ARN`, OIDC flow [5].
6. `Configuring environment variables for the AWS CLI` (same page, `AWS_PROFILE` entry) — "Specifies the name of the AWS CLI profile… overrides the behavior of using the profile named `[default]`… You can override this environment variable by using the `--profile` command line parameter" [6].
7. `Filtering output in the AWS CLI` — https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-filter.html — JMESPath `--query`, server-side `--filter` vs client-side `--query`, cross-service drift example [7].
8. `Setting the output format in the AWS CLI` — https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-output-format.html — `json|yaml|yaml-stream|text|table|off`, the `text`-ordering warning, `--output off` for CI [8].
9. `Inconsistency with Text Output Format` — https://github.com/aws/aws-cli/issues/2418 — concrete example of per-service `--output text` divergence (cloudformation vs ec2) [9].

## 13. Cross-links

- Compares directly to `research/external/non-crypto/gh-cli.md` (envelope, `--jq` vs `--query`) and `research/external/non-crypto/stripe-cli.md` (scoped credentials, plugin model).
- Surfaces requirements around multi-identity CLI posture (A1, A2, A5 in `analysis/01-actor-model.md`).
- Feeds `analysis/04-axes.md` on the "uniform envelope" and "documented precedence" axes.
