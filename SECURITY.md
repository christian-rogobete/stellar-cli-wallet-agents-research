# Security

This repository contains **research and analysis records only**. No executable code, no cryptographic implementations, no deployed service. Reading or citing this material cannot directly introduce a vulnerability into a running system. The security-relevant content consists of:

- **Threat-model analysis** (`analysis/02-threat-model.md`) naming ten threat classes (T1-T10) an AI-agent wallet on Stellar must defend against.
- **Anti-pattern discussion** across the external research files (`research/external/**`) — named defects in shipped systems, with primary-source citations to the public repositories and documentation where each defect is observable. These are cited to inform design choices, not to enable exploitation; every anti-pattern named here is already public in the source it cites.
- **Licence and supply-chain observations** — notably the OpenZeppelin Relayer Channels Plugin AGPL-3.0-vs-MIT-`package.json` discrepancy (documented with three primary sources in `analysis/00-context.md` §5.4) and the gap-list against Meridian Pay's `_auth_contexts` inspection (`research/external/stellar-ecosystem/meridian-pay.md`).

## Reporting issues in the analysis

If you find a **factual error, stale citation, or unsafe recommendation** in the research or analysis record — in particular, one that could lead a downstream implementer astray — please open a GitHub issue or pull request with the file path, section, and a primary source. The repository's revision posture is in `analysis/10-recommendation.md` §9: trivial corrections need no ceremony; material changes produce a new version with a dated amendment log.

If your finding concerns an **undisclosed vulnerability in a third-party project** cited here (for example, a bug in `stellar-cli`, OpenZeppelin `stellar-contracts`, Freighter, or any other referenced system), please **do not file an issue on this repository**. Contact the upstream project through its own coordinated-disclosure channel. This repository's author is not the maintainer of any cited third-party system and cannot accept disclosures on their behalf.

## Scope of trust

The research cites primary sources extensively. Citations are markdown links to the upstream repository or public documentation URL. Readers are encouraged to verify claims against the cited source rather than trust this repository as a secondary. Where a citation has drifted from its upstream (typical for rapidly evolving projects like `openzeppelin-relayer` or `stellar-contracts`), the correct response is to re-read the upstream and update the citation, not to accept the claim in this repository as canonical. Line-number anchors in citations may drift as upstream `main` branches evolve; where line accuracy matters, resolve against the commit hash recorded in the research file's preamble.
