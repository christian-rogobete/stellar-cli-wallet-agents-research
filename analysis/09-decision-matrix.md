# Decision Matrix

**Status:** draft — 2026-04-20
**Audience:** RFP Track delegates
**Depends on:** `00-context.md`, `01-actor-model.md`, `02-threat-model.md`, `03-requirements.md`, `03-requirements-addendum.md`, `04-axes.md`, `06-option-a-cli-first.md`, `07-option-b-agent-native.md`, `08-option-c-layered.md`
**Used by:** `10-recommendation.md`

---

## 1. Method

Options A (CLI-first), B (agent-protocol-first), and C (explicit hybrid) are scored on the eight axes defined in `04-axes.md`:

```
score(option) = sum(axis_weight × axis_score(option) for axis in A1..A8)
```

Scores are integers on a 1-5 scale per `04-axes.md` §3; weights sum to 100; maximum theoretical score is 500. Every cell cites either a section of an option doc (`06` / `07` / `08`), a research file, or a requirement ID.

Non-negotiables are binary gates, not scored axes. All three options are designed to satisfy N1-N6; only scoring differentiates them.

This document:

- Scores every cell individually in §2 with one-line justifications and citations.
- Presents weighted totals in §3.
- Names the three decisive axes (largest contribution to the A-vs-B gap) in §4.
- Runs sensitivity analysis in §5: how fragile is the winner under weight changes.
- Records what the score **does not capture** in §6 (considerations that belong in the recommendation doc, not here).
- States the winner and the honest margin in §7.

## 2. Per-axis scoring

Citations use short forms: `06 §N` = `analysis/06-option-a-cli-first.md` section N; `07 §N` = option B; `08 §N` = option C. Research files cited by path + §. Requirement IDs reference entries in `03-requirements.md` + `03-requirements-addendum.md`.

### 2.1 Axis A1 — Upfront cost to first usable demo (weight 15%)

| Opt | Score | Justification |
|---|---|---|
| A | **5** | Reuses `soroban_cli` public Rust library crate for key derivation, identity files, Ledger signer, XDR plumbing, Soroban tx assembly; ships as a `stellar-<name>` external-binary plugin of the incumbent `stellar-cli`. First usable demo on the order of weeks (`06 §10`). No specification phase before a binary exists. |
| B | **2** | Requires ~20-30 weeks of specification + code-generator + conformance-suite work before a usable CLI demo (`07 §10`). The tool-call protocol schema is the canonical artefact; every surface derives from it, so nothing ships until the schema + generator are both operational. |
| C | **1** | Worse than B: twin-specification requires producing two canonical artefacts, each with its own conformance suite, before any surface is usable (`08 §3.1`). No named reference architecture shows this shipping at RFP-team scale. |

Axis A1 spread: 4 (max).

### 2.2 Axis A2 — Schema-evolution discipline (weight 10%)

| Opt | Score | Justification |
|---|---|---|
| A | **3** | CLI is canonical; MCP is derived through shared dispatch with clap-generated schemas (Kraken-pattern, `research/external/crypto/kraken-cli.md` §9). Drift between the CLI's hand-crafted surface and the derived MCP shape is process-preventable with CI, not structure-prevented. Honest self-assessment in `06 §12` concedes this as a weakness. |
| B | **5** | Canonical schema is the single source of truth; CLI, MCP, and library are code-generated or derived. Drift is structurally prevented: feature addition goes to the schema, regenerates all surfaces, conformance suites run automatically (`07 §3.1`). This is B's headline strength. |
| C | **1** | Twin-specification is the textbook failure mode for schema discipline: two canonical artefacts with no derivation, drift tracked only by human review. `08 §3.1` is explicit that this is the structural failure option C is stated to reject. |

Axis A2 spread: 4 (max).

### 2.3 Axis A3 — Agent-surface ergonomics (non-MCP callers) (weight 15%)

| Opt | Score | Justification |
|---|---|---|
| A | **3** | MCP server ships alongside the CLI from the same dispatch. Non-MCP agent harnesses (Python with custom tools, LangGraph, bespoke Go agents) shell out to the CLI and parse `--json` envelopes. The CLI's strict envelope + exit codes make parsing reliable but it is still shell-out, not typed library access. Honestly flagged in `06 §12` as A's largest weakness vs. B. |
| B | **5** | First-class library bindings (Rust, TypeScript) plus MCP server plus CLI, all derived from the canonical schema. Non-MCP callers import the library directly — typed RPC, no shelling out. This is B's central ergonomic claim (`07 §3.2`). |
| C | **3** | Compose-from-primitives shape has a library and so matches B; twin-specification and namespace-split shapes either duplicate or split the agent surface. Averaged across the three option-C shapes, ergonomics land at mid. The variance itself is a symptom of C's indecision (`08 §3`). |

Axis A3 spread: 2. Option B leads.

### 2.4 Axis A4 — Human-operator ergonomics (weight 10%)

| Opt | Score | Justification |
|---|---|---|
| A | **5** | CLI is crafted — hand-designed help text, idioms borrowed from `gh` / `stripe` / `aws` CLIs (`research/external/non-crypto/gh-cli.md` §9, `stripe-cli.md` §9, `aws-cli.md` §9), shell completions, per-command flag nuance. Human operator (actor A7, break-glass and attended development) has first-class UX. |
| B | **2** | CLI is derived from the schema; "human operator flag tweaks become spec changes, which is awkward during early iteration" (`07 §12`). Flag additions, alias, and help-text refinement require round-tripping through the schema + generator. |
| C | **3** | Namespace-split shape of C gives humans a canonical CLI (comparable to A); other shapes are weaker. Averaging, option C lands mid (`08 §3.3`). |

Axis A4 spread: 3. Option A leads.

### 2.5 Axis A5 — Conformance testability against SEPs (weight 10%)

| Opt | Score | Justification |
|---|---|---|
| A | **3** | Conformance is claimed via integration tests (run the CLI and MCP against a SEP-43 / SEP-48 fixture suite). No standard conformance suite runs against the canonical surface — the canonical surface is the CLI dispatch, not a schema. The adapter path is well-defined, but a delegate verifying conformance reads integration-test output, not a schema-match report. |
| B | **5** | Canonical schema admits conformance tests run as standard suites against the schema itself. Pass / fail is unambiguous and externally verifiable (`07 §3.5`). SEP-43 compatibility is a schema property, not an implementation property. |
| C | **1** | Twin-specification means two schemas, neither fully conformant to SEP-43; conformance must be shown twice (`08 §3.1`). The namespace-split shape has one conformant surface but cannot demonstrate conformance on the other side of the split without redundant testing. |

Axis A5 spread: 4 (max).

### 2.6 Axis A6 — Ecosystem fit and adoption path (weight 15%)

| Opt | Score | Justification |
|---|---|---|
| A | **5** | Explicit extension of `stellar-cli` via external-binary plugin pattern and `soroban_cli` library-crate reuse (`06 §3.1`, `research/external/stellar-ecosystem/stellar-cli.md` §9). Deploys as `stellar-agent-wallet` subcommand alongside the incumbent CLI; users already on `stellar-cli` install the wallet without changing their workflow. Rides SEP-43 / SEP-46-48 standards, Wallets Kit module registry, WalletConnect v2. |
| B | **3** | Rides the same standards (SEP-43, Wallets Kit, WalletConnect) and sits alongside SDF+OZ tracks, but ships as a separate binary with its own canonical schema. No direct extension point into `stellar-cli`; users install a new tool. Integration is via published specs rather than incumbent-tool augmentation (`07 §8`). |
| C | **2** | Multi-front maintenance positions C as parallel to both the incumbent CLI tooling and the agent-spec work. No single clean narrative for ecosystem adoption (`08 §3`). |

Axis A6 spread: 3. Option A leads.

### 2.7 Axis A7 — Maintenance cost at RFP-team scale (weight 15%)

| Opt | Score | Justification |
|---|---|---|
| A | **4** | One canonical surface (CLI dispatch), MCP derived by the same binary. Feature ships once; MCP adapter costs nominal CI attention (Kraken pattern, `research/external/crypto/kraken-cli.md` §9). Not a 5 because the MCP adapter is still real surface to maintain, and human-CLI ergonomics require periodic crafting. |
| B | **3** | Canonical schema + derived adapters; after the generator matures, feature adds are low-cost. But during the RFP-funded window, the generator itself is scope; maintenance during delivery is not linear. Averaged over a 2-3 year horizon, low; during delivery phase, high. Net: 3. |
| C | **1** | Twin-specification is the textbook high-cost shape: every feature ships twice, documents twice, tests twice, deprecates twice (`08 §3.1`). Not sustainable at RFP-funded team scale. |

Axis A7 spread: 3. Option A leads.

### 2.8 Axis A8 — Drift / rot risk on 2-3 year horizon (weight 10%)

| Opt | Score | Justification |
|---|---|---|
| A | **3** | CLI is canonical; drift between CLI and MCP is process-managed. Stellar-protocol evolution (26 "Yardstick" voting 2026-05-06, 27 in testnet thereafter) and SEP updates propagate through the CLI + shared dispatch; MCP updates follow via CI. Survives well with active maintenance; rot-vulnerable under neglect. |
| B | **5** | Canonical schema is rot-resistant by structure. Version bumps rewrite derived surfaces mechanically. SEP updates land in the schema; every surface regenerates to match (`07 §3.1`). Protocol-26 or Protocol-27 CAPs that affect schema shape propagate without hand-editing derived artefacts. |
| C | **1** | Twin-specification is the rot-vulnerable failure mode: two canonical surfaces diverge from SEP-43 independently, and by year 2-3 the two surfaces disagree on what the wallet implements (`08 §3.1`). |

Axis A8 spread: 4 (max).

## 3. Weighted totals

Per-axis weighted contributions: `axis_weight × axis_score`. Maximum per cell is `weight × 5`; total maximum is 500.

| Axis | Weight | A | B | C | A contrib | B contrib | C contrib |
|---|---|---|---|---|---|---|---|
| A1 Upfront cost | 15 | 5 | 2 | 1 | 75 | 30 | 15 |
| A2 Schema discipline | 10 | 3 | 5 | 1 | 30 | 50 | 10 |
| A3 Agent ergonomics | 15 | 3 | 5 | 3 | 45 | 75 | 45 |
| A4 Human ergonomics | 10 | 5 | 2 | 3 | 50 | 20 | 30 |
| A5 Conformance testability | 10 | 3 | 5 | 1 | 30 | 50 | 10 |
| A6 Ecosystem fit | 15 | 5 | 3 | 2 | 75 | 45 | 30 |
| A7 Maintenance cost | 15 | 4 | 3 | 1 | 60 | 45 | 15 |
| A8 Drift risk | 10 | 3 | 5 | 1 | 30 | 50 | 10 |
| **Total** | **100** | | | | **395** | **365** | **165** |
| **% of max (500)** | | | | | **79.0%** | **73.0%** | **33.0%** |

**Winner by weighted sum: option A, by 30 points (6.0 percentage points) over option B. Option C is ~230 points behind and out of contention.**

## 4. The three decisive axes

The axes that most shape the A-vs-B gap (by axis-weight × spread, signed toward A):

| Rank | Axis | A | B | Spread | Signed contribution to A-lead |
|---|---|---|---|---|---|
| 1 | A1 Upfront cost (15%) | 5 | 2 | +3 | +45 |
| 2 | A3 Agent ergonomics (15%) | 3 | 5 | -2 | -30 |
| 3 | A6 Ecosystem fit (15%) | 5 | 3 | +2 | +30 |

A1 and A6 pull toward A; A3 pulls toward B; A1 is the largest single contributor. The decision sits on whether *upfront delivery and ecosystem continuity* outweigh *typed-library access for non-MCP agent harnesses*. Under the weights in `04-axes.md` §5, they do, by 30 points. A different weighting that emphasises agent-ergonomics over delivery speed reverses the answer (see §5 below).

Other axes contribute more modestly:

| Axis | Contribution to A-lead |
|---|---|
| A2 Schema discipline | -20 |
| A4 Human ergonomics | +30 |
| A5 Conformance testability | -20 |
| A7 Maintenance cost | +15 |
| A8 Drift risk | -20 |

Net: A4 + A7 add +45 toward A; A2 + A5 + A8 subtract -60. Combined with the top three (+45 net for A), the total A lead is +30.

## 5. Sensitivity analysis

How fragile is the winner under weight changes?

### 5.1 Single-axis ±5 shifts

`04-axes.md` §5 states the matrix is "robust to ±5 point shifts" on any single axis. Verified:

Max signed-swing per ±5 shift on a single axis:
- A1 (+5 means A loses 5×3 = 15): if A1 weight drops to 10 (and 5 points redistribute into an average axis), A's lead becomes ~30 − 15 = 15. Still A.
- A3 (+5 means B gains 5×2 = 10): if A3 weight rises to 20 (redistribution draws from an average axis), A's lead becomes ~30 − 10 = 20. Still A.
- A8 (+5 means B gains 5×2 = 10): same result.

**No single ±5 axis shift flips the winner.**

### 5.2 Coordinated shifts

A 10-point weight shift from A1 into A3 (a reviewer who believes "agent-ergonomics dominates upfront cost"):

- A1 drops from 15 to 5 → A loses 10×3 = 30 points.
- A3 rises from 15 to 25 → B gains 10×2 = 20 points.
- Net: A's lead of +30 becomes **B's lead of +20**.

**A 10-point weight shift from A1 into A3 flips the winner.**

This is the named alternative weighting: a reviewer who reads "Stellar AI-**agent** wallet" as emphasising the agent side over the Stellar-ecosystem-delivery side would produce a B-winning matrix. The recommendation doc (§7 below; `10-recommendation.md` in full) addresses this reframing honestly rather than dismissively.

### 5.3 Flipping against option C

Option C is 230 points behind A and 200 behind B. No realistic axis re-weighting brings C into contention. The largest single axis is 15% × max spread 4 = 60 points; redistributing every weighting in C's favour produces a gain of at most ~100 points, insufficient to close the gap. **Option C is not a contender under any plausible weighting.** This confirms the conclusion in `08 §5`.

## 6. What the score does not capture

The matrix measures comparable, scoreable axes. It does not capture:

### 6.1 RFP narrative fit

Option A's narrative — "extend `stellar-cli` with an agent-wallet plugin + MCP + policy + smart-account delegation" — is a single-sentence story that delegates can repeat. Option B's narrative — "specify a canonical tool-call protocol and derive CLI, MCP, and library from it" — is more abstract and harder to demo quickly. RFP narrative fit is a real consideration that surfaces in §12 of each option doc but is not an axis.

### 6.2 Team fit and language choice

Option A's extension of `stellar-cli` commits the core to Rust (soroban_cli is Rust). Option B's schema-first approach is language-neutral; any host language is viable. Team fit with Rust vs (say) TypeScript is a real discussion and is not scored.

### 6.3 Positioning vs. SDF + OZ collaborations

`00-context.md` §5.4 flags the SDF + OpenZeppelin collaborations (x402-on-Stellar / x402-MCP; LaunchTube → OZ Relayer). Option B, by specifying a canonical tool-call protocol, **could align with x402-MCP's schema** (or MPP's JSON Schemas, once the MPP spec stabilises beyond its current `-01` individual-submission draft) and become a natural complement. Option A, by extending `stellar-cli`, aligns with ecosystem continuity but is less obviously a schema-complement to either x402-MCP or MPP. Both position defensibly; the difference is rhetorical more than technical. Not an axis.

**Client-SDK ecosystem alignment.** Two Apache-2.0-licensed client SDKs for OZ `stellar-accounts` v0.7.1 already exist (`research/external/stellar-ecosystem/smart-account-kit.md`): kalepail's TypeScript SAK (web-only) and the Kotlin-Multiplatform `OZSmartAccountKit` that Soneso maintains as part of `kmp-stellar-sdk` (Android / iOS / macOS / Web). Both shorten the companion-UI delivery path for either option by providing reference implementations of the smart-account-management half. Disclosure: one of the RFP respondents (Soneso) maintains the KMP SDK; this is noted rather than leveraged — the analysis does not recommend one SDK over the other, and the wallet-owned approval-channel logic that makes a companion *wallet-owned* is additional work either way. Under option B's schema-first framing these SDKs serve the same role, as the companion becomes a schema-projection UI that composes them.

### 6.4 Upstream-dependency risk

Option A's plugin approach assumes `stellar-cli`'s external-binary convention remains stable. If SDF ever deprecates or constrains the plugin pattern, option A absorbs that risk. Option B, by shipping its own binary, does not. This is a small but named risk and is not scored in A6.

### 6.5 Reviewer subjectivity on A3 / A4

A3 (agent ergonomics) and A4 (human ergonomics) are partially subjective. A reviewer who deploys into Python / LangGraph stacks weights A3 higher than one who deploys into MCP-first stacks. The sensitivity analysis above shows the answer changes under a 10-point A1→A3 shift. No axis score here is "wrong"; the weights may be.

## 7. Winner and honest margin

**Winner: option A (CLI-first with agent affordances), by 30 weighted points (79.0% vs 73.0%).**

Honest framing:

- The win is durable against single-axis ±5 shifts and against any realistic re-weighting of option C into contention.
- The win is not commanding. A coordinated 10-point weight shift from A1 into A3 flips the result, a weighting a reviewer could reasonably advocate.
- Option B is a credible second choice; it beats A on four axes (A2, A3, A5, A8) that all concern long-horizon and agent-ergonomics properties.
- Option C is not in contention, confirming that the "explicit hybrid" is a stated-and-rejected alternative, not a third-way finalist.

The recommendation doc (`10-recommendation.md`) will:

- Name option A as the pick.
- Articulate the counterargument from the reviewer who would prefer B, and address it honestly rather than dismiss.
- State the conditions under which the choice should be revisited (a change in team scale, a shift in SEP evolution pace, a change in actor-mix toward non-MCP-first harnesses).
- Name the specific `06-option-a-cli-first.md` components that are load-bearing and must not be silently dropped in implementation.

## 8. Cross-references

- Non-negotiables and context: `00-context.md`.
- Actor model (drives the A3/A4 subjective weight in §6.5): `01-actor-model.md`.
- Threat model (behind A2, A5, A8): `02-threat-model.md`.
- Requirements: `03-requirements.md` + `03-requirements-addendum.md`.
- Axes and weights: `04-axes.md`.
- Options being scored: `06-option-a-cli-first.md`, `07-option-b-agent-native.md`, `08-option-c-layered.md`.
- Recommendation (draws on this matrix): `10-recommendation.md`.
- External research underlying most citations: `research/external/_summary.md`.
- Stellar-capabilities research underlying A2, A5, A6: `research/stellar-capabilities/_summary.md`.

## 9. Revision policy

This matrix is re-scorable. Any change to an axis score or weight requires:

- A one-line comment in the relevant §2.X subsection citing new evidence or a named change in interpretation.
- A re-computation of §3 totals.
- A re-check of §4 decisive axes and §5 sensitivity.
- A note in §7 if the winner or margin changes.

Axis score changes without a cited reason should not be accepted. Weight changes must come from `04-axes.md` §5 (not edited in isolation here).
