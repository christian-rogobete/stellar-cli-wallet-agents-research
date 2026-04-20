# stellar-cli-wallet-agents-research

Open research and analysis records for a self-custodial AI-agent wallet on Stellar, prepared as supporting material for the Stellar Community Fund (SCF) **RFP Track** item *"CLI wallet for agents"* (Q2 2026, led by SDF).

This repository is **not** an RFP specification. The SDF RFP lead owns the requirements. The material here exists to help RFP Track delegates scope and evaluate the item. It surveys the prior art, records non-negotiables, and works through the centre-of-gravity decision (CLI-first vs. agent-native) in enough depth that delegates can compare bidder submissions against a shared baseline.

## Who this is for

**RFP Track delegates** preparing to scope or vote on the *CLI wallet for agents* item. The research is Stellar-native: SEPs, Soroban, sequence numbers, multisig, and sponsored reserves are assumed. Familiarity with agent runtimes (LLM tool-calling, MCP) at a working level but not as a domain.

The material is also dedicated to the public domain under CC0 1.0 and available for anyone who finds it useful, but it was written with delegates as the intended reader.

## About this analysis

The argued recommendation in [`analysis/10-recommendation.md`](analysis/10-recommendation.md) is the operator's researched opinion, not an independent panel finding. The analysis was prepared with research and analysis assistance from Claude Code Opus 4.7. Competing analyses and counter-arguments via pull request are actively invited.

## What is here

```
analysis/          Numbered analysis records (00-10): actors, threats,
                    requirements, three architectural options, decision
                    matrix, recommendation
research/
  external/         20 external wallet / CLI references (14 tier-1 in
                    research/external/, 6 tier-2 in research/external/_tier-2/)
  stellar-capabilities/
                    11 Stellar-capability files (00-10, including MPP)
                    backing the analysis, plus _summary.md
  brain-dump/       The operator's original informal brain-dump,
                    preserved as a research artefact (not a specification)
```

Each research stream uses a shared evaluation template — `research/external/_template.md` for external candidates and `research/stellar-capabilities/_template.md` for Stellar capabilities — so contributors adding a new entry follow a consistent shape.

Research files (`research/external/`, `research/stellar-capabilities/`) are structured around that neutral template and are meant to be reusable as a baseline regardless of architecture. The analysis layer (`analysis/`) is argued: it draws conclusions from the research under a stated set of non-negotiables and weightings.

Start at `analysis/00-context.md` for orientation. The short path for a delegate is:

1. `analysis/00-context.md` — non-negotiables N1-N6; adjacent SDF + OpenZeppelin collaboration tracks; Protocol 26 tracking.
2. `analysis/01-actor-model.md` — seven actors (A1-A7) the wallet serves.
3. `analysis/02-threat-model.md` — fourteen threat classes (T1-T14).
4. `analysis/10-recommendation.md` — the argued recommendation, with the honest counter-case and conditions for revisiting.

Delegates who want the full record read `00` through `10` in order.

## Recommendation

The argued recommendation in [`analysis/10-recommendation.md`](analysis/10-recommendation.md) is **option A: CLI-first with agent affordances**, with the honest counter-case for the schema-first option B addressed directly rather than dismissed. The decision turns on team-scale fit, ecosystem continuity, and the Kraken-pattern precedent for process-enforced schema discipline at comparable scale. It is a researched recommendation, not a consensus finding; §3 of `10-recommendation.md` states the strongest case for the alternative, and §5 names the conditions under which the recommendation should be revisited.

## Licensing

Text content in this repository is dedicated to the public domain under **CC0 1.0 Universal** (`LICENSE`). Reuse (verbatim quotation, translation, derivative works, incorporation into other documents or proposals) is welcome with no attribution required.

Prepared by Christian Rogobete (Soneso), with research and analysis assistance from Claude Code Opus 4.7. RFP Track supporting reviewer for the *CLI wallet for agents* item (Q2 2026).

Source-repository citations in the research files reference upstream projects by short name (e.g. `stellar-cli`, `kraken-cli`, `stellar-contracts`). See `SOURCES.md` for the full mapping to upstream URLs.

## Security-relevant content

The research contains threat-model analysis, anti-pattern discussion, and named defects in shipped systems (with primary-source citations). See `SECURITY.md` for the disclosure posture and how to report issues in the analysis itself.

## Status and scope

**Status.** Draft, 2026-04-20. Content is analytical and may evolve as RFP scoping progresses and as the Protocol 26 mainnet vote (2026-05-06) resolves the CAP-0073 tracking item.

**Scope.** Research and analysis records only. No implementation code lives in this repository.

## Feedback

Issues and pull requests are welcome: corrections, additions, citations, counter-arguments. For substantive changes to the recommendation itself, see §9 ("Revision policy") of `analysis/10-recommendation.md`: trivial corrections need no ceremony; material changes produce a new version with a dated amendment log.
