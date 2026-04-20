# Option C: Explicit layered / hybrid

**Status:** draft — 2026-04-19
**Audience:** RFP Track delegates
**Depends on:** `00-context.md`, `01-actor-model.md`, `02-threat-model.md`, `03-requirements.md`, `06-option-a-cli-first.md`, `07-option-b-agent-native.md`, `research/external/_summary.md`, `research/stellar-capabilities/_summary.md`

---

## 1. Definition

Option C is the **explicit hybrid**: neither CLI nor tool-call protocol is canonical; both are first-class, independently specified, independently versioned, and independently maintained. No derivation. Each surface stands on its own.

This is the option that sounds like the diplomatic middle when someone first hears the CLI-first-vs-agent-native framing. It is stated here, in full, so the decision matrix in `09-decision-matrix.md` can score it honestly against options A and B rather than leave "hybrid" as an unspoken alternative a later reviewer could invoke to question the result.

Option C is **not** the claim that options A and B each have a secondary surface. Both do. Option A ships an MCP server over the CLI dispatch path; option B ships a CLI derived from the tool-call protocol schema. Option C is the specific, distinct claim that **there is no source of truth**: both surfaces are authoritative and neither is derived.

## 2. Three concrete shapes option C could take

Three distinct architectures could all call themselves "explicit hybrid." Naming them specifically forces precision; "layered" otherwise becomes a slippery word that means whatever the speaker has in mind.

### 2.1 Twin-specification

The CLI command tree (argv, help text, exit codes, `--json` output shapes) and the MCP tool schema (tool names, arg schemas, result shapes, annotations) are maintained as two separate canonical artefacts. Each is specified in its own reference document, each has its own conformance suite, each evolves on its own release cadence. Neither references the other as authoritative.

### 2.2 Compose-from-primitives

A shared core library (`wallet-core` or similar, likely Rust) exposes wallet primitives: key derivation, policy evaluation, Soroban auth-entry assembly, SEP-10 / SEP-45 clients, signer dispatch. CLI and MCP are two independent adapters on top; both live in the same monorepo, both are first-class consumers, neither derives from the other. The core is an implementation detail, not a canonical surface.

### 2.3 Namespace split

The surface is partitioned by actor. CLI is canonical for **human operations** — account management, configuration, key generation, recovery, debug, inspection — and is not exposed through MCP. MCP is canonical for **agent operations** — pay, swap, simulate, subscribe-events, request-quote, submit-policy — and has no CLI form beyond a thin pass-through for debugging. A hard split at the command/tool level.

## 3. Why each shape fails as a standalone choice

### 3.1 Twin-specification is structurally unstable

With two canonical artefacts and no derivation, **drift is inevitable**. New features land in whichever surface the writer is most comfortable with; bug fixes to one surface's shape do not propagate automatically; error envelopes become slightly different between `--json` output and MCP tool results; new argument validators get added to the CLI and forgotten in MCP, or vice versa. Every release becomes a reconciliation exercise.

The stream-2 research on SEP-43 / SEP-46 / SEP-47 / SEP-48 sharpens this failure. Those standards describe point-in-time schemas: the canonical Stellar wallet interface, the contract-metadata format, the contract-interface-discovery rules, the contract-interface-spec rules. They evolve. A CLI drifting from its MCP schema means eventually one of them no longer conforms to SEP-43 while the other does; the wallet's conformance story fractures into two. Stream-2 `_summary.md` "Cross-cutting finding 1" relies on the wallet presenting a single typed contract surface; twin-specification breaks that claim.

The **maintenance cost** is the clearest argument. Twin-specification requires writing every feature twice, documenting it twice, testing it twice, and deprecating it twice. For a project with RFP-budget team scale (not enterprise), this is simply not affordable.

### 3.2 Compose-from-primitives is option B in disguise

If CLI and MCP are both projections of a shared core library, then the **core's types, error taxonomy, and schema are the canonical artefact**, not the CLI or the MCP. That is option B. The only difference from option B as stated in `07-option-b-agent-native.md` is terminology: instead of "tool-call protocol," the canonical surface is called "the wallet-core Rust library API." The conformance story is the same (projection of the core); the evolution discipline is the same (change the core, regenerate the surfaces); the upfront cost is the same (specify the core precisely before writing adapters).

The non-crypto design references make this clear. `gh`, `stripe`, and `aws` CLIs are all, internally, projections of a core client library; nobody calls them hybrids. They are CLIs with a core underneath. Calling the core "canonical" is a genuine architectural commitment that `07-option-b-agent-native.md` makes. Compose-from-primitives is a cleaner rendering of option B, not a third option.

### 3.3 Namespace split collapses in practice

The actor model (`01-actor-model.md` §3) rejects the premise. Actor usage crosses the boundary constantly:

- **A2 (multi-agent orchestrator)** uses CLI to provision sub-agents and MCP to coordinate them. Sub-agent provisioning and orchestration are part of the same mental task.
- **A4 (user-facing assistant with human approval)** uses MCP in the agent path and CLI in the companion approval path. The approval UX is one user-facing flow with two surfaces underneath.
- **A7 (human operator, break-glass, attended development)** uses the CLI for recovery and for attended development with IDE agents, and switches between the CLI and MCP constantly inside a single working session — an IDE agent proposes commands over MCP, the human inspects and runs them at the terminal, then queries audit log state from the CLI. The split cannot be symmetric in the recovery direction either: A7 must understand what the MCP surface did in order to recover from it.

Worse: any operation genuinely needed in both namespaces forces a de-facto canonical. If `pay` is canonical-in-MCP and the CLI also wraps it for humans, the CLI `pay` is derived in everything but name. If `accounts create` is canonical-in-CLI and agents need to create accounts too, the MCP `accounts_create` tool is derived. The question "which is canonical?" is not avoided by namespace split; it is muddied, and each operation is answered differently without consistent principle.

## 4. What option C gets right (and what it does not fix)

To be honest: every option A or option B implementation is **layered in practice**. Option A ships an MCP server over the CLI dispatch (Kraken pattern, `research/external/crypto/kraken-cli.md` §9). Option B ships a CLI derived from the schema. **Layering is inevitable; the choice is which layer is the source of truth.**

Option C's core insight — that both surfaces matter — is correct. But option C's only distinct architectural claim is the refusal to pick a source of truth. That refusal has costs (drift, conformance fracture, duplicated work) without compensating benefits under this analysis record's actor model and team scale. The insight is right; the answer is wrong.

## 5. When option C would be justified

Option C is defensible only if all three of the following hold:

1. **The two surfaces serve genuinely disjoint audiences** with effectively zero overlap. The actor model for this project has extensive overlap (§3.3).
2. **The two surfaces have distinct feature sets** that do not need to compose. The features here compose extensively: typed preview of a Soroban invocation must look the same whether a human approves via CLI or an agent renders it via MCP.
3. **The two surfaces can be maintained by separate teams indefinitely.** The RFP-funded team, at RFP-stage and likely post-award, is small; one authoritative surface is operationally cheaper by a large margin.

None of the three conditions hold for the Stellar AI-Agent Wallet. Option C is therefore not viable in its pure form for this project. (It could be viable in some hypothetical future fork where an enterprise spin-out maintains the CLI for sysadmins while a separate product team maintains the agent protocol. That is a different project with a different team, not this RFP.)

## 6. Why option C is stated anyway

The explicit hybrid is stated here for three reasons beyond fairness:

1. **Fairness to future readers.** The CLI-first-vs-agent-native framing produces a rhetorical pull toward compromise. Without option C stated explicitly, a later reviewer asks "why didn't you consider a hybrid?" and the analysis record has no articulated answer. The record is stronger for having the hybrid on paper and the rejection explicit.

2. **Terminology discipline.** Without option C as a named counter-option, "hybrid" becomes a slippery word used to mean whatever the speaker has in mind. Naming the three concrete shapes — twin-specification, compose-from-primitives, namespace split — forces precision in every future discussion.

3. **Self-check against scope creep during implementation.** Implementing option A or option B honestly means declining to silently turn into option C over time. If a year into implementation an implementer finds themselves maintaining a second canonical artefact (a parallel spec for MCP that does not derive from the CLI dispatch, or a hand-written CLI that does not project from the tool-call protocol), the project has quietly migrated to twin-specification without the decision having been revisited. This document is the anchor reviewers point back to when a PR proposes "just one more canonical surface."

## 7. How the decision matrix scores option C

`09-decision-matrix.md` scores option C against the same axes as options A and B. Expected result (to be verified when the matrix is computed):

- **High on** flexibility (both surfaces are first-class by construction), ecosystem compatibility (each surface can be made conformant to its idioms independently), and team-size agnosticism (teams can work on separate surfaces in parallel).
- **Low on** operational cost (features shipped twice, docs twice, conformance suites twice), schema-evolution discipline (no structural defence against drift), conformance-testing strength (two surfaces diverge from SEP-43 independently), upfront-time-to-usable-demo (both surfaces need first-release polish), and maintainability-at-RFP-scale (smallest team that can sustain twin-specification is larger than the RFP-funded team).

The matrix's weighted axes should reflect the project's actor model and non-negotiables. Under that weighting, option C is expected to trail both options A and B. The interesting question for the recommendation doc is not "does option C win" but "**by how much** does it lose, and to which of A or B." That gap is informative: if C loses by a small margin, the A/B choice is under-determined; if by a large margin, the A/B choice is what matters and C is a sanity-check.

## 8. Cross-references to the companion options

- `06-option-a-cli-first.md` §2 argues that the MCP surface is legitimately **derived** from the CLI dispatch path (via Kraken-pattern shared schema generation), not a second canonical artefact. That is **option A's rejection of option C**.
- `07-option-b-agent-native.md` §2 argues that the CLI is legitimately **derived** from the tool-call protocol (via schema-driven code generation), not a second canonical artefact. That is **option B's rejection of option C**.

Both rejections are honest. Each identifies option C's failure mode (duplication without compensating benefit) and proposes its own canonical layer. The decision matrix in `09` compares A vs B on which canonical layer fits the actors, threats, and non-negotiables better, with C as the scored-but-declined alternative.

## 9. What this option explicitly does not do

Option C, as stated, does not:

- Replace the requirements in `03-requirements.md`. Most requirements are satisfied identically by any of the three options; only a handful are sensitive to the centre-of-gravity choice (conformance-testing requirements, schema-evolution requirements, specific MCP or CLI ergonomics requirements).
- Pre-empt the decision matrix. Expected scores above are guidance for the matrix authors, not binding.
- Foreclose a future migration. If team scale grows, the actor model changes, or SEP-43's evolution pace slows, option C may become defensible. Decisions of this class should be revisited annually; a reason for revisiting is explicitly not "someone re-read this doc and preferred the sound of hybrid."

## 10. Primary risks of stating option C this way

- **Strawman risk.** A reviewer may read §3 and believe this document has constructed a weak version of option C specifically to reject. Mitigation: the three shapes (twin-specification, compose-from-primitives, namespace split) are the three architecturally distinct versions surfaced in prior art (Kraken-CLI for Kraken-style CLI-first; MoonPay OWS for MoonPay-style schema-first; Coinbase CDP for namespace-style agent-vs-admin surfaces). They are not straw; they are the market-observable shapes.

- **Under-stating the maintenance cost.** Enterprise teams do maintain twin-specification successfully (AWS CLI is close to this, per `research/external/non-crypto/aws-cli.md` §9-§10). The rejection here is scoped to RFP-team-scale; at enterprise scale the verdict could differ.

- **Missing a novel shape.** The three shapes in §2 are the ones surfaced in prior art; a future reviewer may propose a fourth. If so, the right response is to add it to §2 and evaluate honestly, not to reject without articulation.

## 11. Cross-references

### Requirement citations

Option C's rejection rests on requirements not yet stable under the reconciliation pass. Stable citations will be added when `03-requirements.md` reconciliation completes. Expected rejection anchors include:

- Requirements on schema-evolution discipline (conformance-testing against SEP-43 and adjacent standards).
- Requirements on maintenance cost (single canonical specification; single conformance suite).
- Requirements on approval-UX cohesion across CLI and MCP (a single rendering of a Soroban invocation for human approval regardless of surface).

### Research citations

- **Drift as a maintenance burden**: `research/external/non-crypto/aws-cli.md` §10 (per-service output drift). AWS CLI demonstrates that twin-like surfaces drift even under institutional maintenance.
- **Single-dispatch-two-transports pattern**: `research/external/crypto/kraken-cli.md` §9, arguing against twin-specification by example; one dispatch path, two transports.
- **Schema-driven surfaces**: `research/external/crypto/moonpay-agents-ows.md` §9 (seven-sub-spec modularisation with declared conformance). A single canonical schema produces verifiable conformance that twin-specification cannot match.
- **Attestation registry convergence**: `research/stellar-capabilities/_summary.md` "Cross-cutting finding 10": the same architecture appears from three directions (Safe ERC-7484, MetaMask Snaps audit pinning, OZ stellar-contracts verifier contracts). A single canonical surface claims this convergence cleanly; twin-specification fractures it.

---

## Conclusion

Option C is the honest explicit hybrid. Stated here to foreclose the "why not a hybrid" question; rejected in the recommendation doc on operational-cost and actor-overlap grounds. Neither of those arguments is ideological; each rests on this RFP's specific scope, team scale, and actor model. A future project with a different shape may reasonably choose differently.

The real decision lives in `09-decision-matrix.md` between options A and B. Option C's role here is to make that decision visible by standing adjacent as the declined alternative.
