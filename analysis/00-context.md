# CLI-first vs. Agent-native: Context, Scope, and Non-Negotiables

**Status:** draft — 2026-04-20
**Audience:** Stellar RFP Track delegates
**Part of:** centre-of-gravity analysis for the *CLI wallet for agents* RFP item

---

## 1. Purpose

This folder contains the analysis record for one question:

> Should the *CLI wallet for agents* on Stellar be designed **CLI-first with agent affordances**, or **agent-protocol-first with a CLI surface**?

**Option A — CLI-first with agent affordances.** The canonical surface is a command-line tool, shipped, for example, as a plugin of the incumbent `stellar-cli`. The MCP server for agents is a derived transport over the same command-dispatch path; a human operator uses the same CLI directly. Adding a feature means editing the CLI command tree; MCP schemas are generated from that tree.

**Option B — agent-protocol-first with a CLI surface.** The canonical surface is a typed tool-call protocol (JSON Schema per tool, versioned, conformance-testable). A CLI, a library crate, and an MCP server are all derived projections of that schema. Adding a feature means editing the schema; every transport regenerates from it.

A third option, an "explicit hybrid" where neither surface is derived from the other, is stated and rejected as option C in `08-option-c-layered.md`.

The two approaches produce superficially similar artifacts but diverge in threat surface, policy model, and adoption path. The choice is load-bearing: it shapes every downstream question (custody, key types, SEP coverage, extensibility, packaging).

The analysis is captured across a set of short, numbered documents rather than one monolithic file, so sections can be revised and cited independently.

## 2. Audience

**RFP Track delegates** reading the research to prepare for scoping or evaluating the *CLI wallet for agents* RFP item. Assumed Stellar-native: SEPs, Soroban, sequence numbers, multisig, and sponsored reserves do not need introduction. Assumed familiar with agent runtimes (LLM tool-calling, MCP) at a working level but not as a domain.

Delegates are busy. Documents favour signal over surface area.

## 3. What this analysis covers

In scope:

- The wallet's **centre of gravity**: which surface is primary and which is derived.
- The **trust model for the caller**: is the caller treated as an agent by default, or as a script operated by a human?
- **Policy locus**: where spending limits, intent validation, and capability checks live.
- **Primary interfaces** and how they relate (CLI, MCP, library import, companion UI).

Out of scope for *this* document (each is addressed elsewhere in the repo and summarised here so readers know where to look):

- **SEP coverage and order of implementation** — `research/stellar-capabilities/05-seps.md`; `analysis/03-requirements.md` §1.5 (`REQ-sep-*`); `analysis/03-requirements-addendum.md` §5.5.
- **Smart-account contract choice** — resolved to OpenZeppelin `stellar-accounts` pinned at exact version `= 0.7.1` (`REQ-sa-oz-baseline`). See `research/stellar-capabilities/04-smart-accounts.md` and `analysis/03-requirements-addendum.md` §5.4.
- **Key storage backends per platform** — OS keyring first (`REQ-acct-keyring-first`), hardware signer (`REQ-acct-hardware-signer`), external-signer plugin (`REQ-acct-signer-plugin`). See `analysis/03-requirements.md` §1.1.
- **Language / runtime of a reference implementation** — implicit Rust core under option A (reuses the `soroban_cli` public library crate); option B likewise Rust core with generated bindings. See `analysis/06-option-a-cli-first.md` §3.1 and `analysis/07-option-b-agent-native.md` §3.2.
- **Packaging, distribution, and release cadence** — `analysis/03-requirements.md` §2.3 (`REQ-pkg-*`, `REQ-plat-*`).

## 4. Non-negotiables

These are constraints, not differentiators. Any viable option must satisfy all of them.

| # | Constraint | Rationale |
|---|---|---|
| N1 | **Self-custodial.** Keys never leave the user's or agent's host without explicit, per-action user consent. | Core property of the product; prerequisite for RFP alignment with Stellar's ecosystem values. |
| N2 | **Autonomous operation.** The wallet must function with no dependency on a project-operated backend. | Avoids vendor lock-in and a central point of failure/compromise. |
| N3 | **No central server holding user credentials, keys, policies, or transaction history.** Telemetry, if any, is opt-in and non-identifying. | Follows from N1/N2; also a privacy property. |
| N4 | **Open source, permissive licence.** | RFP requirement; enables ecosystem adoption. |
| N5 | **Deterministic, scriptable output.** JSON is the default machine interface; human-readable rendering is a derived view. | Agents cannot reliably screen-scrape. Determinism is a correctness property, not a nicety. |
| N6 | **Testnet and mainnet parity.** Every capability works on both, with clear network scoping. | Prevents the common class of bugs where testnet success does not imply mainnet success. |

N1-N3 rule out some popular "agent wallet" designs from outside the Stellar ecosystem (cloud-resident session keys, server-side policy engines, custodial agent accounts). The decision below is between two approaches that both satisfy these constraints.

## 5. External reference candidates

Each candidate is evaluated under `research/external/` against a shared template (`research/external/_template.md`) and summarised in `research/external/_summary.md`. Candidates that pass the template's non-negotiable check become direct design references; the rest remain useful as partial references or counter-examples.

### 5.1 Tier-1 reference set (13 candidates)

- **Crypto, agent-oriented.** Kraken CLI (`krakenfx/kraken-cli`); MoonPay Agents + Open Wallet Standard (`github.com/open-wallet-standard`); Trust Wallet Agent Kit (TWAK); Coinbase Agentic Wallet / CDP Wallets; Turnkey; Safe + modules.
- **Crypto, Stellar ecosystem.** Meridian Pay (SDF-built smart wallet); Stellar MCP server (`christopherkarani/stellar-mcp`); Freighter + Wallet Kit; `stellar-cli`.
- **Non-crypto, agent-CLI design.** `gh`, `stripe`, `aws` CLIs.

### 5.2 Tier-2 reference set (6 candidates)

Stored under `research/external/_tier-2/`: Privy, Fireblocks, MetaMask Snaps, Tether WDK, LaunchTube, Phantom MCP server. Second-wave candidates surfaced from tier-1 findings and useful for specific patterns (Fireblocks vocabulary, MetaMask Snaps skill model, LaunchTube submission primitives, Phantom MCP commit-step anti-pattern, Tether WDK annotation discipline, Privy token-scope model). Cited from the analysis documents where relevant.

### 5.3 Protocols the wallet interoperates with (not design references)

- **x402** — Coinbase-origin HTTP 402-based agent-to-service payments. See §5.4.
- **MPP (Machine Payments Protocol)** — HTTP-402 agentic-payment protocol from Tempo and Stripe, submitted to the IETF Datatracker as individual submission `draft-ryan-httpauth-payment` (at `-01`). Stellar-supported via `@stellar/mpp` SDK (MIT). Covered in `research/stellar-capabilities/10-mpp.md`. Coexists with x402 on the same 402 response (different headers).
- **SEP-10, SEP-24, SEP-45, and other relevant SEPs** — handled in `research/stellar-capabilities/05-seps.md`.

### 5.4 Adjacent SDF + OpenZeppelin collaborations that frame the RFP

The Stellar Development Foundation has entered a **pattern of collaboration with OpenZeppelin** on agent-adjacent infrastructure. At least two tracks are active or announced as of 2026-04; any submission against this RFP item will need to position relative to both. The pattern itself (not just either track in isolation) is what delegates are likely to have in mind.

**Track 1: x402-on-Stellar + x402-MCP server** (early 2026). Lets AI agents discover paid resources, authorise payments via smart wallets, and chain paid API calls under user-defined spending policies. Sources: `stellar.org/blog/foundation-news/x402-on-stellar`, `developers.stellar.org/docs/build/agentic-payments/x402`. Tier-1 external research confirmed this is **separate from Meridian Pay** (zero `x402` references across both SDF repos for Meridian Pay). The x402-MCP server is a distinct track.

**Track 2: LaunchTube → OpenZeppelin Relayer (Channels Plugin).** The existing SDF-operated LaunchTube service is being **discontinued** in favour of OpenZeppelin Relayer with the Channels Plugin. This affects every Stellar-side project that currently depends on LaunchTube for transaction submission (Meridian Pay via its wallet-backend, the Stellar MCP server, and others). Tier-2 external research confirmed the deprecation via `developers.stellar.org` pages.

Consequences for this RFP:

- **Neither track can be ignored.** Delegates will compare submissions to SDF's in-flight efforts both individually and collectively.
- **Neither track's scope matches the RFP item directly.** Track 1 is about paying for API resources; track 2 is about transaction submission infrastructure. A self-custodial agent-wallet primitive includes x402 as one flow and must provide an alternative to LaunchTube-class submitters (see below). These are positioning angles, not refutations.
- **Licence on track 2 — AGPL-3.0 based on available evidence, with a caveat.** Verified 2026-04-18: the `LICENSE` file in `github.com/OpenZeppelin/relayer-plugin-channels` contains the verbatim GNU AGPL v3 text; GitHub's licence-detection API returns `agpl-3.0`; the parent `github.com/OpenZeppelin/openzeppelin-relayer` repo and the `@openzeppelin/relayer-sdk` npm package both declare AGPL-3.0-or-later. However, the Channels Plugin repo's `README.md` and `package.json` `"license"` field say `MIT`, which contradicts the `LICENSE` file. By OSS convention the `LICENSE` file is authoritative, but the discrepancy leaves the effective licence an open vendor-clarification item (see `research/external/_tier-2/stellar-ecosystem/launchtube.md` §11 and `research/stellar-capabilities/08-infra-ops.md` §7). If AGPL-3.0 is the effective licence, it conflicts with N4 as a deployment dependency, and a permissively-licensed design needs an alternative submitter: the in-process SEP-5-derived channel-account pool is the self-custodial candidate.
- **Expected downstream artifacts.** Each option doc (`06-option-a-cli-first.md`, `07-option-b-agent-native.md`) documents the delta to both tracks' agent-facing surfaces. `research/stellar-capabilities/08-infra-ops.md` documents the track-2 migration and the in-process alternatives.

A second research stream documents Stellar capabilities themselves — what the wallet can build on — under `research/stellar-capabilities/`. Files in this folder cite both streams by file path + anchor.

### 5.5 Protocol 26 ("Yardstick"): tracked but not yet binding

Protocol 26 is on testnet since **2026-04-16**; mainnet vote scheduled **2026-05-06**. Tracked CAPs per the preambles in [`stellar-protocol/core/`](https://github.com/stellar/stellar-protocol/tree/master/core) (accessed 2026-04-19): **CAP-0077** (ledger-key freeze via network config), **CAP-0078** (TTL-extension v2 with min/max), **CAP-0079** (muxed-address strkey host functions), **CAP-0080** (ZK BN254 host functions), **CAP-0081** (TTL-ordered eviction), and **CAP-0082** (checked 256-bit int math).

**CAP-0073 status is ambiguous.** Its own preamble declares `Protocol version: TBD`, not P26. However, it is grouped with the P26 bundle in `core/README.md` in the same repo. Whether it ships in P26 at the 2026-05-06 mainnet vote, or slips to a later protocol, is an open item. Tracked until the vote resolves it.

None of the P26 bundle (with or without CAP-0073) invalidates the non-negotiables (N1-N6) or the option A / option B architectural choice. The one CAP that touches a requirement is CAP-0073: if it ships, SAC-initiated trustline creation and auto-create of G-accounts become possible from inside a Soroban call, which means a wallet's minimum-reserve guard must inspect Soroban footprints for reserve-mutating side effects, not only classic `ChangeTrust` / `CreateAccount` / sponsored-reserve ops. This is a one-sentence update to the relevant requirement, deferred until the vote resolves CAP-0073's inclusion. CAP-0077 adds a `FROZEN_ENTRY` failure reason that belongs in wallet error-mapping; not a design change. See `research/stellar-capabilities/08-infra-ops.md` for tracking.

## 6. Document map

| File | Content |
|---|---|
| `00-context.md` | This document. |
| `01-actor-model.md` | Who and what uses this wallet. Drives requirements. |
| `02-threat-model.md` | Threats, trust boundaries, framing. Drives architecture. |
| `03-requirements.md` | MUST / SHOULD / NICE, mapped to actors. |
| `03-requirements-addendum.md` | Compact-form patch log (36 merges, 2 splits, 106 adds — 86 stream-2 + 12 MPP + 3 smart-account additions from the OZ v0.7.1 review + 5 smart-account additions from the SAK + KMP `OZSmartAccountKit` review, 2 promotions, 1 candidates-queue resolution). |
| `04-axes.md` | Axes of comparison and their weights. |
| `06-option-a-cli-first.md` | Concrete shape of the CLI-first design. |
| `07-option-b-agent-native.md` | Concrete shape of the agent-native design. |
| `08-option-c-layered.md` | The honest hybrid, stated explicitly as a third option. |
| `09-decision-matrix.md` | Scores and justifications. |
| `10-recommendation.md` | Argued recommendation, counterarguments addressed, open questions. |

The numbering preserves a gap at `05` where a dedicated Soroban / SEP surface-per-option document was originally planned. The underlying material — Soroban auth-entry flows, the SEP-43/46/47/48 coverage, the `__check_auth` pattern — is distributed across `06-option-a-cli-first.md` §7, `07-option-b-agent-native.md` §7, and the Stellar-capabilities research files under `research/stellar-capabilities/` (notably `03-soroban.md`, `04-smart-accounts.md`, and `05-seps.md`).

Files `00` through `02` are prerequisites. Files `03` onward depend on them and are written in order.

## 7. Glossary (scoped to this document set)

- **Agent** — an LLM-driven process that issues tool calls. May run attended (human in the loop) or unattended. Assumed hostile by default (see `02-threat-model.md`).
- **Agent runtime** — the harness that hosts the LLM and dispatches tool calls. Examples: Claude Code, OpenAI Agents SDK, LangGraph, MCP clients.
- **Center of gravity** — the primary interface from which other interfaces are derived. A decision about what the wallet "is," not just what it exposes.
- **Policy engine** — the component that evaluates a proposed transaction against rules (limits, allowlists, time windows) before signing.
- **Pre-auth / auth entries** — Soroban `AuthorizationEntry` / `SorobanAuthorizationEntry` flows, where a signer authorises a specific invocation out of band.
- **Self-custodial** — the user, or a process operating on the user's host under the user's control, holds the signing keys; no third party can sign.
- **Smart account (C-account)** — Soroban contract acting as an account (e.g. OpenZeppelin's stellar-contracts account). Enables programmable auth and policy on-chain.
