# Recommendation

**Status:** draft — 2026-04-20
**Audience:** RFP Track delegates
**Depends on:** `00-context.md`, `01-actor-model.md`, `02-threat-model.md`, `03-requirements.md`, `03-requirements-addendum.md`, `04-axes.md`, `06-option-a-cli-first.md`, `07-option-b-agent-native.md`, `08-option-c-layered.md`, `09-decision-matrix.md`

---

## 1. The pick

**The analysis recommends option A: CLI-first with agent affordances.** The wallet would be built as a self-contained agent CLI binary, shipped as `stellar-agent-wallet` (working name) under the incumbent `stellar-cli`'s external-binary plugin convention, reusing the `soroban_cli` public Rust library crate for key derivation, identity files, Ledger signer, XDR plumbing, and Soroban transaction assembly. A built-in MCP server over stdio is a derived transport over the same dispatch path (Kraken pattern), not a separate specification. The local policy engine, smart-account delegation via OZ `stellar-accounts` context rules and policies, in-process SEP-5-derived channel-account pool, and SEP-43 + Wallets Kit + WalletConnect v2 interoperation are all first-class components of the option-A architecture as drafted in `06-option-a-cli-first.md`.

This choice is recorded in `09-decision-matrix.md` with a weighted-sum score of 79.0% (option A) versus 73.0% (option B) and 33.0% (option C), under the weights specified in `04-axes.md` §5.

## 2. Why option A wins under the stated weighting

Option A wins on four axes and loses on four. The wins are concentrated on the axes the RFP weighting prioritises: stage, team scale, and ecosystem fit.

- **A1 Upfront cost to first usable demo (15% weight, +3 spread → +45 points).** Reusing `soroban_cli` and shipping as a `stellar-<name>` plugin produces a signing binary in weeks, not months. The alternative path (option B) spends 20-30 weeks specifying a canonical tool-call schema plus building a code generator before any surface is usable.
- **A6 Ecosystem fit and adoption path (15% weight, +2 spread → +30 points).** Option A is an **extension of the incumbent Stellar CLI**, not a rival artefact. Users who already install `stellar-cli` add the wallet as a subcommand; Freighter and the Wallets Kit module registry integrate through SEP-43 unchanged; `stellar-cli` users do not have to learn a second command tree.
- **A7 Maintenance cost at RFP-team scale (15% weight, +1 spread → +15 points).** One canonical surface with a derived MCP transport is sustainable for a 2-4-engineer team. Twin-specification (option C) and schema-plus-generator (option B during the delivery window) are not.
- **A4 Human-operator ergonomics (10% weight, +3 spread → +30 points).** The crafted CLI — hand-designed help text, idiom-match to `gh` / `stripe` / `aws`, per-command flag nuance, shell completions — is the surface actor A7 (human operator, break-glass) depends on. A derived CLI (option B) forces flag tweaks through a schema + generator round-trip.

The losses are real but narrower:

- **A3 Agent-surface ergonomics (15% weight, -2 spread → -30 points).** Non-MCP agent harnesses (Python with custom tool protocols, LangGraph, bespoke Go agents) shell out and parse JSON rather than importing a typed library. Option B is structurally better here; option A compensates with strict JSON envelopes, predictable exit codes, and field-selection syntax (`gh`-pattern), but compensation is process, not structure.
- **A2 Schema-evolution discipline, A5 Conformance testability, A8 Drift risk (10% weights each, -2 spreads → -20 each = -60 total).** Option B's canonical schema protects these axes structurally; option A protects them through CI, deprecation windows, and shared-dispatch convention. Process rather than structure.

Net: +120 on wins, -90 on losses, +30 net lead.

## 3. The honest case for option B

A careful reviewer can argue option B should win. The argument is not weak and deserves direct engagement rather than dismissal.

**The B case in one sentence.** A wallet called "Stellar AI-**agent** wallet" whose canonical surface is a CLI — rather than an agent-facing tool-call schema — puts agents second by construction, and hides structural drift under process discipline that a small team will eventually fail to sustain.

**The B case in more detail.** Three points:

1. **Agent ergonomics are the wallet's central use case, not its secondary surface.** The non-MCP portion of the agent market — Python agents, LangGraph, OpenAI Agents SDK, bespoke Go / Rust harnesses — is substantial and growing. Option A treats these as second-class (shell out, parse JSON); option B treats them as first-class (import library, call typed functions). A wallet that optimises for agent use should not make agents the second-class surface.
2. **Schema-evolution discipline is a 3-year property, not a 3-month property.** SEP-43, SEP-46, SEP-47, SEP-48 are live standards that will evolve. Protocol 26 votes 2026-05-06; Protocol 27 follows; SEP revisions are ongoing. A wallet whose canonical surface is a schema evolves with the schema by construction; a wallet whose canonical surface is a CLI evolves through process, and process fails under small-team pressure. The RFP horizon is 1-2 years funded; sustainability is 3+ years.
3. **The SDF + OZ x402-MCP track is schema-centric.** The announced x402-on-Stellar + x402-MCP server (`00-context.md` §5.4) is likely a schema-first artefact. A wallet that ships its own canonical schema (option B) complements x402-MCP by composing schemas. A wallet whose canonical surface is a CLI (option A) complements x402-MCP by invoking it (a less tight integration).

All three points are correct observations. The matrix registers them: option B scores 5 on A3 (agent ergonomics), 5 on A2 (schema discipline), 5 on A5 (conformance testability), 5 on A8 (drift risk). It is not a weak option; it is a strong option under a different weighting.

**The specific weighting that flips the matrix.** A 10-point weight shift from A1 (upfront cost) into A3 (agent ergonomics) makes B the winner by 20 points instead of A by 30. A reviewer who believes delivery speed should matter less and agent-ergonomics should matter more is not making a mistake; they are making a judgment about the project's centre of gravity.

## 4. Why the analysis still picks option A given the B case

The B case is real but is outweighed on the specific constraints recorded in `00-context.md` §4 and `04-axes.md`. Three grounds:

### 4.1 Delivery at RFP-team scale is load-bearing, not optional

The RFP envelope assumes a 2-4-engineer team. That is the binding constraint on option B's upfront cost. A 20-30-week specification + generator phase before a usable artefact exists is not a timeline a 2-4-engineer team sustains with confidence; schedule slip in the generator phase produces a wallet that does not exist. Option A ships a working artefact in weeks and refines from there. The RFP is funded against delivery, not against an abstract architectural ideal.

This is not an argument that B is technically worse; it is an argument that A fits the RFP team-scale constraint. Under an enterprise-scale team (10+ engineers) the constraint relaxes and B's upfront-cost disadvantage shrinks. That team is not the assumed implementer.

### 4.2 Ecosystem continuity is a first-class RFP property

Stellar is a small ecosystem. Users already install `stellar-cli`. Extending an incumbent tool rather than shipping a rival tool is not a narrative preference — it is a credibility statement about how the wallet positions relative to the ecosystem. A reviewer asking "why should Stellar users install a new wallet instead of what they already have?" has a ready answer under option A: the tool is already installed; the wallet ships as a plugin.

Option B's narrative requires users to adopt a new binary with a different command tree. The cost is real even if its magnitude is uncertain.

### 4.3 Process-enforced schema discipline is sufficient for the RFP horizon under the Kraken pattern

Option B's schema discipline advantage is structural; option A's is process-enforced. Process discipline has a failure mode — neglect — but the Kraken CLI's shared-dispatch + clap-derived MCP schema has been stable in production (`research/external/crypto/kraken-cli.md` §9). The pattern is proven at comparable scale and time horizon.

The B case's strongest argument — that small teams fail at process discipline — is a real risk but not an unmanaged one: CI catches drift, Kraken shows the pattern is sustainable, and the alternative (B's generator) is itself scope that a small team must sustain. One maintenance burden is traded for another, not removed.

### 4.4 The B case's best claims are partially captured by A anyway

Option A ships an MCP server; its agent ergonomics are not zero. The gap vs. option B is "typed library import" vs "shell out to a CLI with strict JSON envelope". Real, but narrow. Most 2026 agent harnesses are MCP-first; the Python/LangGraph cohort that benefits from a typed library is substantial but not the majority.

On schema evolution: option A ships SEP-43 + SEP-46 + SEP-47 + SEP-48 adapters as first-class components (per the requirements addendum). The difference vs. option B is that A's adapters are hand-written and tested via integration, while B's adapters are generated. Both produce conformance; both survive SEP evolution with active maintenance.

## 5. Conditions under which the recommendation should be revisited

The recommendation is not permanent. If any of the following change, the decision should be rerun:

1. **Team scale grows to 6+ engineers capable of sustaining two canonical surfaces.** A7 score for B rises; B's upfront-cost penalty amortises over more people. Threshold: an RFP award that funds 6+ FTE for 18+ months, or a post-award scale-up event.
2. **Stellar RFP runway extends 6+ months.** A1 spread narrows. Same effect as team growth.
3. **Actor mix shifts toward non-MCP-stack agents dominating.** A3 weight rises materially. Threshold: usage telemetry (opt-in, per N3) shows >60% of wallet invocations from non-MCP agent harnesses.
4. **SEP-43 / SEP-46-48 revisions accelerate.** A2 and A8 weights rise. Threshold: more than two significant SEP revisions per year requiring schema re-adaptation.
5. **SDF + OZ x402-MCP publishes a canonical schema that becomes an extension point, or MPP reaches IETF consensus with a formal Stellar channel spec.** Either event turns option B's approach into "extend the published schema" rather than "specify a new one from scratch." This reduces B's upfront cost and gives it the ecosystem-fit property A currently claims. Thresholds: a published, stable x402-MCP tool-call schema with conformance tests; or the MPP core draft (`draft-ryan-httpauth-payment`) reaching WG-track status plus a published `draft-stellar-channel-*` in `mpp-specs`.
6. **A named security incident in the Kraken-pattern shared-dispatch model** (in Kraken or elsewhere) that demonstrates the process-enforced drift risk is realisable. Threshold: any CVE or public post-mortem.

If none of these happen, the recommendation stands.

## 6. Load-bearing components of option A

These are the components of `06-option-a-cli-first.md` a successful implementation would need to preserve. If any are removed or watered down without the decision being revisited, the wallet quietly becomes something else: most likely option B in disguise without option B's discipline, or a twin-specification drift into option C.

1. **External-binary plugin pattern extending `stellar-cli`.** The wallet ships as `stellar-<name>` on `PATH`, discovered by the incumbent CLI. If this extension point becomes unavailable, A6 (ecosystem fit) collapses and the extension narrative that underlies option A's positioning falls away.
2. **`soroban_cli` public Rust library-crate reuse.** Key derivation, identity files, Ledger signer, XDR plumbing, Soroban tx assembly, SEP-5 HD paths are reused, not re-implemented. Reimplementation shifts A1 from +45 to ~0 and takes the upfront-cost advantage with it. Requirement anchors include `REQ-acct-keyring-first`, `REQ-acct-derivation-sep05`, `REQ-sor-simulate-first`.
3. **MCP server over shared dispatch** (one command tree, two transports; schemas generated from clap metadata; the Kraken pattern). Not a separate specification. If the MCP server becomes a parallel artefact with its own tool schema, option A has silently become option C's twin-specification.
4. **In-process policy engine as a Rust component inside the binary.** Policy is local, not vendor-side (N2 / N3). Key requirements: `REQ-sec-policy-per-tx-cap`, `REQ-sec-policy-per-period-cap`, `REQ-sec-policy-counterparty-allowlist`, `REQ-sec-policy-rate-limit`, `REQ-sec-policy-minimum-reserve`, `REQ-sep-41-policy`, `REQ-svc-mpp-policy-gating` (MPP has no human-in-the-loop gate in its spec; local policy is the sole defence for MPP charge and channel flows). If policy moves to a separate process or to the agent's prompt, the T2 / T4 defences collapse.
5. **Smart-account delegation via OZ `stellar-accounts` context rules plus policies** — not Safe-style modules, which OZ does not expose. `REQ-sa-oz-baseline` (MUST, pinned to `= 0.7.1`), `REQ-sa-no-relayer-hard-dep`, `REQ-sa-policy-selector-scope`, `REQ-sa-atomic-signer-threshold-update`, `REQ-sa-verifier-pinning`, `REQ-sa-context-rule-caps`, `REQ-sa-policy-initiated-exec` (reframed: custom wallet machinery for recurring-payment and top-up flows, not an OZ-provided feature — OZ `Policy::enforce()` cannot originate transactions). OZ context rules bind signers and policies to specific `Context` types with an optional `valid_until`; the caller explicitly selects which rule governs each auth context, and the v0.7 auth digest `sha256(signature_payload ‖ context_rule_ids.to_xdr())` binds those rule IDs into the signed payload, structurally closing the rule-ID-downgrade vector (T11). If delegation reverts to key-sharing or to off-chain config, or if the wallet composes Safe-shaped modules the OZ account contract does not expose, T4 defences collapse.
6. **In-process SEP-5-derived channel-account pool.** `REQ-classic-sequence-pool` (promoted to MUST; see `03-requirements-addendum.md` §2). Parallel submission without vendor relayer. If this is replaced by a hard dependency on OZ Relayer, N4 (permissive licence) conflicts with AGPL-3.0; if removed entirely, actors A1 (automation daemons) and A2 (multi-agent orchestrators) cannot scale.
7. **SEP-43 adapter + Wallets Kit module + WalletConnect v2 interoperation including `signAuthEntry` and `signMessage`.** `REQ-sep-sep43-walletkit`, `REQ-sep-24-handoff`. The wallet integrates into every Stellar dapp already using Wallets Kit; omitting this is a strategic self-inflicted wound.
8. **Wallet-issued nonce for commit step** (the fix to Phantom's advisory-only `confirmed: false → true` pattern, from `research/external/_tier-2/crypto/phantom-mcp-server.md` §9; applies to both x402 and MPP charge flows). If the commit step is an agent-passed boolean, T2 (prompt-injected transactions) defences collapse.
9. **Chain-namespaced tool names with CAIP-2 on every schema.** `REQ-ux-mcp-tool-names-namespaced`, `REQ-sec-network-required`. Structural T10 defence at the tool boundary.
10. **Uniform `{ok, data|error, request_id}` envelope** across every CLI command and MCP tool. `REQ-ux-json-envelope`. One parse path for agents; no per-command drift into per-service formats (the AWS CLI anti-pattern documented in `research/external/non-crypto/aws-cli.md` §10).
11. **Agentic-payment-protocol support for both x402 and MPP.** `REQ-svc-x402-consume`, `REQ-svc-mpp-charge` (MUST), `REQ-svc-mpp-channel` (SHOULD), `REQ-svc-mpp-policy-gating` (MUST), `REQ-svc-mpp-x402-coexistence` (SHOULD). Omitting either protocol leaves actor A3 under-served; omitting MPP specifically leaves an IETF-track standard (`draft-ryan-httpauth-payment`) unsupported. See `research/stellar-capabilities/10-mpp.md`.

Any implementation plan that compromises one of these eleven without a revised recommendation silently changes what the wallet is.

## 7. What this recommendation does not foreclose

The recommendation names the architectural centre of gravity. It does not foreclose:

- **Implementation language beyond the Rust-core implication.** `soroban_cli` is Rust; the core wallet logic is therefore expected to be Rust. UI layers (companion app) can be anything. The MCP transport layer can be Rust or a thin adapter.
- **A specific MCP tool taxonomy.** The tool schema is derived from the CLI's clap dispatch; the per-tool structure is an implementation detail. Relevant surface-per-option material is covered in `06-option-a-cli-first.md` §7 and `07-option-b-agent-native.md` §7.
- **A specific smart-account contract.** The recommendation names OZ `stellar-accounts` pinned at exact version `= 0.7.1` as the baseline (the library is pre-1.0 and the `v0.7+` floating range is not acceptable); forking, extending, or replacing the baseline is an implementation decision with separate trade-offs.
- **A ship date or v1 feature scope.** Project-planning is a separate track; this doc names the architecture, not the calendar.
- **A position on SEP-31.** Currently OUT pending confirmation by an implementer; the recommendation does not pre-empt that call.
- **A specific AMM venue choice for v1.** `research/stellar-capabilities/07-dex-amm.md` §7 names three candidates; the recommendation does not select among them.
- **Choice of companion-UI SDK composition.** `REQ-sa-companion-sdk-composition` names SAK (web), KMP `OZSmartAccountKit` (native multiplatform), or both as the reference implementations the companion builds on; the mix is an implementation decision and may differ for desktop, mobile, and web surfaces.

## 8. What an implementer would tackle first

In rough order of priority:

1. **Implementation plan.** A separate document naming phases, milestones, which components ship first, which tracks can run in parallel. The handoff artefact from research to engineering.
2. **Full-format expansion of the 106 new requirements** recorded in compact form in `03-requirements-addendum.md` (86 stream-2 + 12 MPP + 3 smart-account additions from the OZ v0.7.1 review + 5 smart-account additions from the SAK + KMP `OZSmartAccountKit` review). Documentation task; does not block code.
3. **Resolve the 18 open follow-ups** from `research/stellar-capabilities/_summary.md` §Open follow-ups. Most are small (licence clarifications, verify an address, confirm a roadmap item). Prioritise the OZ Relayer Channels Plugin `README.md` / `package.json` vs `LICENSE` discrepancy with OpenZeppelin.
4. **Protocol 26 CAP-0073 requirement update** to the minimum-reserve guard after the 2026-05-06 mainnet vote confirms the final bundle.
5. **Track the MPP channel-spec draft.** No `draft-stellar-channel-*` exists yet in `mpp-specs`; `REQ-svc-mpp-channel-dispute` and the SHOULD-level `REQ-svc-mpp-channel` are blocked on this spec landing. When it publishes, re-evaluate both requirements and the default-store question for commitment keys.
6. **Vendor clarifications.** OZ Channels Plugin licence status (`research/stellar-capabilities/08-infra-ops.md` §7); StellarExpert Open API licence (`research/stellar-capabilities/06-tooling.md` §7); MPP IETF draft identifier alignment (the repo file is `draft-httpauth-payment-00.md` but the IETF Datatracker entry is `draft-ryan-httpauth-payment` at `-01` — follow-up with the MPP working group to understand the stabilisation path).
7. **Ecosystem-contribution proposals.** Three surfaced in research: channel-account SDK helpers (relevant to any Stellar client SDK adding an agent-wallet integration path); SEP-30 recovery-server model; per-counterparty muxed-account interop survey.

## 9. Revision policy

This recommendation is the terminal document of the analysis record for this architectural choice. It is not re-edited casually; substantive changes require either:

- A documented trigger from §5 (conditions to revisit), or
- A formal re-score of `09-decision-matrix.md` producing a different winner, or
- A named change to `04-axes.md` weights producing a different winner.

Trivial edits (typos, link updates, pointer corrections) do not need ceremony. Everything else is a material change and should produce a new version with a dated amendment log.

## 10. Cross-references

- Non-negotiables that bound every option: `00-context.md` §4.
- Actors this wallet serves: `01-actor-model.md`.
- Threats this wallet defends against: `02-threat-model.md`.
- Requirements the wallet satisfies: `03-requirements.md` + `03-requirements-addendum.md`.
- Axes and weights: `04-axes.md`.
- Option A's concrete architecture: `06-option-a-cli-first.md` (this is the primary implementation reference).
- Option B (the credible second choice): `07-option-b-agent-native.md`.
- Option C (stated and rejected): `08-option-c-layered.md`.
- Decision matrix producing this pick: `09-decision-matrix.md`.
- External research underlying the citations: `research/external/_summary.md`.
- Stellar-capabilities research underlying the implementation plan: `research/stellar-capabilities/_summary.md`.
- SDF + OpenZeppelin collaboration tracks the wallet positions against: `00-context.md` §5.4.
