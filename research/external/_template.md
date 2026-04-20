# Per-Candidate Research Template

**How to use:** Copy the sections below into each candidate file under `research/external/<group>/<slug>.md`. Keep the headings in order so candidates can be diffed and compared. Fields marked *required* must be populated or explicitly marked `unknown — see §12`.

---

# [Candidate name]

**Status:** not yet researched | in progress | complete
**Researcher:** [agent id / human name]
**Last updated:** [YYYY-MM-DD]
**ID:** [slug matching filename]
**Category:** crypto-agent / crypto-stellar / non-crypto

---

## 1. What it is (required)

One paragraph. Product or protocol in plain terms. Avoid marketing language.

## 2. Agent interface (required)

How does a code-driven caller use it? CLI / SDK / HTTP / MCP / IDE extension? One concrete end-to-end example: *the minimum sequence a programmatic caller performs to do a signed action.*

## 3. Custody model (required)

Where do the signing keys live? Who can sign? What is the role of the user's device, the vendor's infrastructure, and the agent host in the signing path? If there are multiple custody tiers, list them.

## 4. Policy and approval model

What rules can the caller express (amount limits, counterparty allowlists, time windows, asset scoping, etc.)? Where are they enforced (client-side, vendor-side, on-chain, none)? Who can change them and through what channel? Does the policy survive vendor compromise, host compromise, or both?

## 5. Delegation model

How is authority granted to a sub-process, sub-agent, or service? How is delegation bounded? How is it revoked, and how fast does revocation take effect? Is delegation expressed as a key share, a capability, a scoped signature, or a policy rule?

## 6. Relevant protocol coverage

- For crypto candidates: chains, account types, SEP (or chain-equivalent) features.
- For non-crypto candidates: auth scheme, idempotency, pagination, error-envelope shape, webhook/event model.

## 7. Licence and adoption signals (required)

- Licence:
- Source available: yes / no / partial
- Last meaningful commit:
- GitHub stars (if OSS):
- Known production users:
- Commercial backing / funding stage (if company):
- Ecosystem traction (integrations, imports, citations):

## 8. Non-negotiable check (required for crypto; n/a for non-crypto)

Against `analysis/00-context.md` §4:

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes / no / partial | |
| N2 | Autonomous (no project-operated backend) | yes / no / partial | |
| N3 | No central server for keys/policy/history | yes / no / partial | |
| N4 | Open source, permissive licence | yes / no | |
| N5 | JSON-default output | yes / no / n/a | |
| N6 | Testnet/mainnet parity | yes / no / n/a | |

Any "no" or "partial" does not disqualify the candidate from being a *design* reference but removes it from being a direct deployment model.

## 9. What to adopt (required)

Concrete design ideas worth taking. One bullet per idea, each specific enough to cite in a requirement. Example: *"Wrap every command's output in `{ ok: bool, data | error, request_id }` envelope — see `aws-cli` §6."*

## 10. What to avoid (required)

Specific design choices in this candidate that fail our threat model, non-negotiables, or actor needs. Each bullet names the choice and the failure.

## 11. Open questions

Things research could not resolve. Each becomes a follow-up task or a known unknown cited in the decision matrix.

## 12. Sources (required)

Primary sources only — official docs, source code, announcements, changelogs. Avoid blog summaries. For each source: URL, accessed date, relevance.

Cite source-file references as markdown links to the upstream GitHub URL, preferably pinned at a commit hash so line numbers remain valid, e.g. [`rs-soroban-sdk/soroban-env-host/src/auth.rs#L123`](https://github.com/stellar/rs-soroban-sdk/blob/<commit-hash>/soroban-env-host/src/auth.rs#L123). Record the commit hash in the preamble of the research file. See `SOURCES.md` at the repo root for the full short-name → upstream URL mapping.

## 13. Cross-links

Requirements this candidate surfaced (by requirement ID once assigned). Other candidates this one compares to directly.
