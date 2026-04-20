# Axes of Comparison

**Status:** draft — 2026-04-20
**Audience:** RFP Track delegates
**Depends on:** `00-context.md`, `01-actor-model.md`, `02-threat-model.md`, `03-requirements.md`, `03-requirements-addendum.md`
**Used by:** `09-decision-matrix.md`, `10-recommendation.md`

---

## 1. Purpose

This document defines the axes along which options A, B, and C are compared in the decision matrix (`09-decision-matrix.md`). Each axis has:

- A one-sentence statement of what the axis measures.
- A rationale for why it matters.
- A 1-5 scoring convention with explicit anchors.
- The requirement IDs that anchor it (so a scorer can verify claims against the requirements).
- A weight as a fraction of 100%.

The axes are **differentiators**: they measure the gap between options after the non-negotiables have already been satisfied. An option that fails a non-negotiable is disqualified, not low-scored; scoring runs only over candidates that clear §4 of `00-context.md`.

## 2. Non-negotiables vs. axes

The six non-negotiables (N1-N6) in `00-context.md` §4 are binary gates. They do not appear in the matrix because every option in this record is designed to satisfy them. Options that fail a non-negotiable are out by definition. They do not score at all.

The axes below measure **how** each option satisfies the non-negotiables and the requirements: a design can be self-custodial and autonomous in a way that is maintainable, extensible, and idiomatic, or in a way that creates structural drift, operational burden, or ecosystem friction. That is what this document scores.

## 3. Scoring convention

Each axis is scored on a 1-5 integer scale. The anchors below are shared across axes; axis-specific refinements appear in §4.

| Score | Meaning | Evidence bar |
|---|---|---|
| 5 | Strong — a named reference architecture demonstrably does this well; the option can adopt it nearly verbatim. | At least two primary-source citations (research files) showing the pattern in production use, plus a clear path to adoption with no named blockers. |
| 4 | Good — the option does this well but requires non-trivial implementation work, or one named reference is only a partial fit. | One strong primary-source citation + one identified implementation blocker that is solvable. |
| 3 | Mixed — the option can do this, but the path is ambiguous or depends on trade-offs whose resolution is not yet agreed. | Citation exists; path is debatable; scoring depends on judgment. |
| 2 | Weak — the option struggles here; compensating mechanisms exist but are not structural. | Citation of the failure mode; named mitigation exists but is process-enforced, not design-enforced. |
| 1 | Poor — the option fails this axis in a way that is hard to mitigate without changing the option's core. | Citation of the failure mode; no named mitigation that preserves the option's identity. |

Scores are integers. Fractional scores are not used. If a judgement sits between two anchors, the scorer picks one and documents the tension in the matrix cell comment.

The matrix is re-scorable. Any change to an axis score requires a one-line comment citing either new evidence or an acknowledged change in interpretation of existing evidence.

## 4. Axis catalog

Eight axes. Each is scored 1-5; weights sum to 100%.

### A1. Upfront cost to first usable demo

**Statement.** How much implementation work is needed before a deployed binary can sign and submit a real Stellar transaction end-to-end.

**Why it matters.** RFP-stage work is cost-sensitive; a credible demo within the award window (typically 8-16 weeks) is load-bearing. Options that require extensive upfront specification before the first usable artefact score lower.

**How to score.**

- 5 — reuse of existing tooling (`soroban_cli` library crate + `stellar-cli` extension plugin pattern) puts a demo on the order of weeks.
- 4 — limited new infrastructure (custom config, MCP adapter); most of the surface is derived or reused.
- 3 — meaningful new infrastructure but with a clear reuse anchor (one major component reused, others specified and implemented).
- 2 — substantial specification + conformance work before any user-facing artefact exists.
- 1 — demo gated on two or more major specification tracks completing in parallel.

**Requirement anchors.** `REQ-ux-mcp-stdio`, `REQ-ux-json-envelope`, `REQ-pkg-prebuilt-binaries`, `REQ-cfg-two-layer`.

**Weight: 15%.**

### A2. Schema-evolution discipline

**Statement.** How the canonical surface handles version drift as features land, deprecations roll out, and SEP standards evolve.

**Why it matters.** SEP-43, SEP-46, SEP-47, SEP-48 are live standards that evolve. A surface that drifts from its canonical schema fails conformance over time. Structural defences outperform process defences on a 2-3 year horizon.

**How to score.**

- 5 — canonical schema is the single source of truth; every other surface (CLI, library, MCP tool) is code-generated or derived; drift is structurally prevented.
- 4 — one canonical surface with a convention-enforced derivation into others (e.g., shared dispatch + metadata); drift is process-preventable with CI.
- 3 — canonical surface exists but derivation is partial (some surfaces are hand-written wrappers).
- 2 — two or more surfaces are maintained in parallel with hand-written conformance; CI must catch drift.
- 1 — two canonical surfaces, no derivation, drift tracked only by human review.

**Requirement anchors.** `REQ-sep-sep43-walletkit`, `REQ-sep-48-typed-preview`, `REQ-sep-47-claim-check`, `REQ-sep-sep10-client`, `REQ-sep-45-client`.

**Weight: 10%.**

### A3. Agent-surface ergonomics (non-MCP callers)

**Statement.** How well non-MCP agent harnesses (Python agents using custom tool protocols, LangGraph graphs, bespoke Node / Go harnesses) can talk to the wallet with typed interfaces, without resorting to shell-out and string parsing.

**Why it matters.** MCP is the dominant agent-tool protocol in 2026 but not the only one. Actor A2 (multi-agent orchestrator) and A3 (service-consumer agent) may run on stacks that are not MCP-aware. Option A is weakest here; option B is strongest; option C is mid.

**How to score.**

- 5 — first-class typed library (Rust / TS bindings) + MCP server; non-MCP callers talk to the library directly without shelling out.
- 4 — MCP server + one other programmatic surface (e.g., stable JSON-RPC over stdio); non-MCP callers have a clear typed path.
- 3 — MCP server only; non-MCP callers shell out but the CLI surface is strict enough (envelope, exit codes, `--json`) to parse reliably.
- 2 — MCP server only; non-MCP callers shell out and parse JSON; some operations not easily scriptable.
- 1 — agents are steered toward MCP by design; non-MCP use is explicitly second-class.

**Requirement anchors.** `REQ-ux-mcp-stdio`, `REQ-ux-mcp-tool-annotations`, `REQ-ux-json-envelope`, `REQ-ux-exit-codes`, `REQ-cfg-sdk-parity`.

**Weight: 15%.**

### A4. Human-operator ergonomics

**Statement.** How well a human operator (actor A7, break-glass and attended development) can discover, invoke, and recover from the wallet's operations at a terminal.

**Why it matters.** Actor A7 is a first-class actor. The wallet must be usable for human recovery even when the agent is off or compromised. Crafted CLI UX (help text, completions, idioms borrowed from `gh` / `stripe` / `aws`) cannot be replaced by a machine-generated interface without loss.

**How to score.**

- 5 — CLI is crafted (hand-designed help text, aliases, completions, idioms); actor A7 prefers it for interactive use.
- 4 — CLI is largely hand-crafted with some machine-assisted sections; discoverability is strong.
- 3 — CLI is usable for humans but generated from schema; idiom-match to `gh` / `stripe` is partial.
- 2 — CLI is functional but obviously derived; humans frequently reach for MCP client or library instead.
- 1 — CLI is last-class; its purpose is scripting, not human operation.

**Requirement anchors.** `REQ-ux-setup-wizard`, `REQ-ux-table-render`, `REQ-ux-companion-ui`, `REQ-ux-horizon-deprecation`, `REQ-classic-op-preview`.

**Weight: 10%.**

### A5. Conformance testability against SEPs

**Statement.** How easily the wallet can demonstrate, in test, that it conforms to SEP-43 (wallet API), SEP-46 (contract meta), SEP-47 (interface discovery), SEP-48 (interface spec), SEP-10/-45 (auth), and related standards.

**Why it matters.** Conformance is how an RFP delegate verifies the wallet does what it claims. A surface that can be conformance-tested by running a standard suite against it scores higher than one that requires bespoke testing per SEP.

**How to score.**

- 5 — standard conformance suites run against the canonical schema; passes or fails are unambiguous and externally verifiable.
- 4 — canonical schema admits conformance tests; suites written in-house but auditable.
- 3 — conformance is claimed via integration tests; no standard suite runs against the canonical surface.
- 2 — conformance is a per-feature assertion, tested through the implementation not through a schema.
- 1 — conformance cannot be demonstrated without reading the code.

**Requirement anchors.** `REQ-sep-sep43-walletkit`, `REQ-sep-48-typed-preview`, `REQ-sep-47-claim-check`, `REQ-sep-sep10-client`, `REQ-sep-45-client`.

**Weight: 10%.**

### A6. Ecosystem fit and adoption path

**Statement.** How naturally the option plugs into the existing Stellar ecosystem (stellar-cli plugin convention, Wallets Kit module registry, WalletConnect v2 methods, operator-maintained SDKs) and how it positions against the SDF + OZ tracks (x402-MCP server, OZ Relayer with Channels Plugin).

**Why it matters.** The RFP is Stellar-native; delegates want to see continuity with what exists, not a parallel ecosystem. Extending an incumbent surface scores higher than shipping a rival surface.

**How to score.**

- 5 — explicit extension of an incumbent tool (`stellar-cli` plugin + `soroban_cli` library crate reuse); deploys as `stellar-agent-wallet` subcommand alongside existing tooling.
- 4 — rides existing conventions (SEP-43, Wallets Kit, WalletConnect) but ships as a separate binary; integration is via published specs.
- 3 — parallel to existing tooling; integration is through typed adapters on top of shared primitives.
- 2 — ships its own conventions; integrations are in the ecosystem's direction but require downstream adopters to learn new APIs.
- 1 — positions as a rival to incumbent tools or to SDF/OZ tracks; ecosystem friction is explicit.

**Requirement anchors.** `REQ-sep-sep43-walletkit`, `REQ-sa-oz-baseline`, `REQ-cfg-sdk-parity`, `REQ-pkg-prebuilt-binaries`, `REQ-sa-no-relayer-hard-dep`.

**Weight: 15%.**

### A7. Maintenance cost at RFP-team scale

**Statement.** How sustainable the ongoing maintenance is for a small team (the RFP-funded team, estimated at 2-4 engineers, not an enterprise-scale org).

**Why it matters.** The RFP pays for delivery of a working wallet. Maintenance after the award runs on operator-team capacity. Options that scale poorly below an enterprise team headcount produce credibility risk.

**How to score.**

- 5 — one canonical surface, one reference implementation, one test suite; maintenance cost scales with features, not surfaces.
- 4 — one canonical surface with a thin derived layer; maintenance is almost linear.
- 3 — canonical surface plus meaningful derived artefacts that need independent tests.
- 2 — two surfaces with one canonical; derived surface still costs real maintenance.
- 1 — twin-specification or similar; every feature costs ~2× to ship, test, document.

**Requirement anchors.** `REQ-classic-sequence-pool` (MUST — in-process submitter), `REQ-sa-no-relayer-hard-dep`, `REQ-cfg-sdk-parity`, `REQ-pkg-prebuilt-binaries`.

**Weight: 15%.**

### A8. Drift / rot risk on a 2-3 year horizon

**Statement.** How well the option's canonical surface survives two to three years of evolution without silent rot: Stellar protocol versions (25 → 26 → 27 and beyond), SEP updates, language-ecosystem churn, agent-tool-protocol shifts (MCP is current; successors are possible).

**Why it matters.** The RFP-funded wallet is not a short-lived demo. It must survive Protocol 26 (voting 2026-05-06), Protocol 27 and beyond, SEP evolution, and shifts in the agent-tool-protocol landscape. Structural defences outperform process defences on this horizon.

**How to score.**

- 5 — rot-resistant by structure; version-bump evolution rewrites derived surfaces but leaves the canonical surface stable.
- 4 — rot-resistant with light process support; upstream Stellar-protocol changes propagate mechanically.
- 3 — rot-resistant with heavy process support; change management is a named operational discipline.
- 2 — rot-sensitive; silent drift is likely between major versions without active review.
- 1 — rot-vulnerable by design; the twin-spec failure mode or equivalent.

**Requirement anchors.** `REQ-sor-policy-contract-pinning`, `REQ-sor-auth-entry-freeze-window`, `REQ-ux-horizon-deprecation`, `REQ-cfg-sdk-parity`, `REQ-sep-48-typed-preview`.

**Weight: 10%.**

## 5. Weighting justification

Weights sum to 100. The distribution reflects three priorities implied by the RFP scope and the RFP-funded team-scale assumption:

1. **Deliverability at RFP-team scale** (A1 15% + A7 15% + A6 15% = 45%). The single largest cluster. A wallet that cannot ship credibly during the award window, or that cannot be maintained after it, does not exist. Ecosystem fit is grouped here because it is how delivery velocity becomes credible: an implementer rides existing conventions instead of inventing.

2. **Agent-use-case fit** (A3 15% + A2 10% + A5 10% = 35%). The second cluster. The wallet exists because agent use cases exist. Ergonomics for non-MCP callers, schema-evolution discipline to keep up with SEPs, and conformance testability against those standards are the three things that make the wallet usable to its primary actors.

3. **Long-term durability + human-operator posture** (A4 10% + A8 10% = 20%). The third cluster. Human-operator ergonomics and rot-resistance are not negligible but are secondary; a wallet that ships and works for agents is the precondition for a wallet that also survives over years.

The weights are not fine-tuned; they are stated as round numbers so the matrix is robust to ±5 point shifts. A reviewer who disagrees with this weighting should restate the weights and rerun the matrix in §10 of `09-decision-matrix.md`. The matrix is a function of the weights, not of a particular reviewer's intuition.

Specific weighting choices worth noting:

- **A3 (agent ergonomics) equal to A1 (upfront cost)**. One could argue A1 should outweigh A3 given RFP scope. The counter-argument: a wallet that ships fast but serves agents poorly is not a Stellar **AI-agent** wallet; the A-word in the name matters. Equal weight is a deliberate balance.
- **A2 (schema discipline) at only 10%**. Lower than A3 because schema discipline protects the option against long-term rot; it is not a day-one differentiator. A reviewer prioritising 3-year horizon would raise this.
- **A4 (human ergonomics) at only 10%**. Lower than A3. The argument is that actor A7 (human operator) is a break-glass actor, not the daily user. A reviewer who disagrees (for example, one who sees sysadmins as the primary buyer) would raise this.
- **A7 (maintenance cost) at 15%**. Tied to team scale. A larger team could absorb more maintenance burden; at RFP-team scale, 15% reflects the risk that an option wins on paper and loses in operation.

## 6. How the decision matrix uses this

`09-decision-matrix.md` produces a single weighted-sum score per option:

```
score(option) = sum(axis_weight * axis_score(option) for axis in A1..A8)
```

Max theoretical score is 500 (5 × sum of weights = 5 × 100). Scores are reported both raw and as percentage-of-max for readability.

The matrix also reports:

- Per-axis scores alongside a one-line justification per cell.
- The three axes with the largest spread between options (the axes that most drive the decision).
- A sensitivity note: if a reviewer re-weights by ±5 on any axis, does the winner change?

The recommendation doc (`10-recommendation.md`) draws on the matrix result plus counterarguments that the matrix cannot score (e.g., "this option positions a submission poorly in RFP narrative," "this option's upstream maintainer cannot absorb the proposed module").

## 7. Revision policy

Axes and weights are not frozen. Revisions are expected when:

- New evidence changes an axis anchor (e.g., Stellar-cli's plugin convention turns out to have a licence clash that was missed).
- A new non-negotiable is identified (lifted from an axis to a gate).
- A reviewer proposes a different weight distribution and argues it.

Every revision to this document records:

- The date.
- A one-line summary of what changed and why.
- A note on whether `09-decision-matrix.md` scores need re-evaluation.

Trivial typo fixes do not need a note. Substantive changes (weight shifts ≥ 5, axis added / removed, scoring convention changed) always do.

## 8. Cross-references

- Non-negotiables (gates): `00-context.md` §4.
- Requirements this document uses to anchor axes: `03-requirements.md` + `03-requirements-addendum.md`.
- Actor model (drives the weighting discussion in §5): `01-actor-model.md`.
- Threat model (behind schema-discipline and drift-risk axes): `02-threat-model.md`.
- Options being scored: `06-option-a-cli-first.md`, `07-option-b-agent-native.md`, `08-option-c-layered.md`.
- Matrix output: `09-decision-matrix.md`.
- Recommendation: `10-recommendation.md`.
