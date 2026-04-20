# Per-Capability Research Template

**How to use:** Copy the sections below into each capability file under `research/stellar-capabilities/<NN>-<slug>.md`. Keep headings in order so capabilities can be diffed and referenced consistently. Write for a reader who is Stellar-native but not necessarily expert in this specific area.

---

# [Capability area]

**Status:** stub | in progress | complete
**Last updated:** [YYYY-MM-DD]
**ID:** [slug matching filename stem, e.g. `03-soroban`]

---

## 1. What it is

Plain-language description **scoped to what an AI-agent wallet needs to know**. Not a Stellar tutorial. Assume the reader knows the ledger, accounts, and SDKs at a working level.

## 2. Components

Concrete operations, protocol elements, SDK surfaces, contracts, endpoints, or on-chain objects in this area. One bullet per component; link to primary docs.

## 3. Relevance to an AI-agent wallet

- **Actors served** (from `analysis/01-actor-model.md`): `A#`, `A#`, ...
- **Non-negotiables touched** (from `analysis/00-context.md` §4): `N#`, `N#`, ...
- **Threats mitigated or created** (from `analysis/02-threat-model.md`): `T#`, `T#`, ...

One short paragraph explaining *why this area matters to the wallet decision*, not just to Stellar in general.

## 4. Agent-specific quirks

Things a programmatic, possibly unattended caller handles that a UI user does not — and vice versa. Examples: concurrent sequence-number contention, fee estimation under load, state-expiration surprises, simulation-before-submit, unit confusions, silent defaults that differ per network. Specific, with reasoning.

## 5. Status on the network

- Mainnet: production / preview / experimental / unavailable / deprecated
- Testnet: same
- Roadmap: pending CAPs, SEPs, or protocol releases affecting this area
- Known incidents or breaking changes in the last 12 months

## 6. Primary documentation

Links to `docs.stellar.org`, relevant CAPs and SEPs, SDK reference (ours and the reference JS/Rust SDKs), contract sources for any smart-contract components. Include access date for each.

Source-code and protocol citations should be written as markdown links to the upstream GitHub URL, e.g. [`stellar-protocol/ecosystem/sep-0010.md`](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0010.md), [`rs-soroban-sdk/...`](https://github.com/stellar/rs-soroban-sdk/blob/main/...), [`stellar_flutter_sdk/...`](https://github.com/Soneso/stellar_flutter_sdk/blob/master/...). See `SOURCES.md` at the repo root for the full short-name → upstream URL mapping.

## 7. Known gaps or pain points

Things that make this capability awkward, risky, or incomplete in an agent context. These often map to open work upstream; record them so the wallet design does not quietly depend on them being fixed.

## 8. Candidate requirements surfaced

Requirements that follow from this capability. Each gets a short tag that will be cited verbatim in `analysis/03-requirements.md`. Format:

- `REQ-<area>-<short-slug>`: one-line statement. Proposed classification (MUST / SHOULD / NICE / OUT). Proposed source tag.

These flow into `analysis/03-requirements.md` via its candidates queue.

## 9. Cross-links

- Related capability files:
- External candidates (under `research/external/`) that interact with this capability:
- Decision files that will cite this entry:
