# Sources

Research files in this repository cite upstream projects by short name (e.g. `stellar-cli`, `kraken-cli`, `stellar-contracts`). This document maps every short name to its upstream URL. Most source-file references in the research files are written as markdown links pointing directly at those upstream URLs; the mapping below is provided so any remaining short-name citations can be resolved to a canonical source.

Research was conducted against local clones or symlinks of each upstream at specific points in time. "Local clone" means a shallow or pinned clone; "symlink" means a pointer to a globally maintained working copy of the same upstream.

## Stellar core and protocol

| Short name | Upstream |
|---|---|
| `stellar-protocol` | https://github.com/stellar/stellar-protocol |
| `rs-soroban-sdk` | https://github.com/stellar/rs-soroban-sdk |
| `rs-soroban-env` | https://github.com/stellar/rs-soroban-env |
| `stellar-rpc` | https://github.com/stellar/stellar-rpc |
| `stellar-cli` | https://github.com/stellar/stellar-cli |

## Smart accounts and passkey

| Short name | Upstream |
|---|---|
| `stellar-contracts` | https://github.com/OpenZeppelin/stellar-contracts |
| `smart-account-kit` | https://github.com/kalepail/smart-account-kit |
| `passkey-kit` | https://github.com/kalepail/passkey-kit |

## Client SDKs (maintained by the author)

| Short name | Upstream |
|---|---|
| `stellar_flutter_sdk` | https://github.com/Soneso/stellar_flutter_sdk |
| `stellar-ios-mac-sdk` | https://github.com/Soneso/stellar-ios-mac-sdk |
| `stellar-php-sdk` | https://github.com/Soneso/stellar-php-sdk |
| `kmp-stellar-sdk` | https://github.com/Soneso/kmp-stellar-sdk |

## Reference client SDKs (third-party)

| Short name | Upstream |
|---|---|
| `js-stellar-base` | https://github.com/stellar/js-stellar-base |
| `js-stellar-sdk` | https://github.com/stellar/js-stellar-sdk |
| `java-stellar-sdk` | https://github.com/lightsail-network/java-stellar-sdk |
| `py-stellar-base` | https://github.com/StellarCN/py-stellar-base |
| `typescript-wallet-sdk` | https://github.com/stellar/typescript-wallet-sdk |

## Ecosystem

| Short name | Upstream |
|---|---|
| `freighter` | https://github.com/stellar/freighter |
| `anchor-platform` | https://github.com/stellar/java-stellar-anchor-sdk |
| `soroban-examples` | https://github.com/stellar/soroban-examples |

## External comparators (agent-wallet and MCP references)

| Short name | Upstream |
|---|---|
| `kraken-cli` | https://github.com/krakenfx/kraken-cli |
| `stellar-mcp` | https://github.com/christopherkarani/stellar-mcp (fork of `JoseCToscano/stellar-mcp`, pinned at 2025-03-31 state) |
| `tw-agent-skills` | https://github.com/trustwallet/tw-agent-skills |
| `smart-wallet-demo-app` | https://github.com/stellar/smart-wallet-demo-app |
| `sdp-backend-meridian-pay` | https://github.com/stellar/stellar-disbursement-platform-backend (branch `meridian-pay`) |
| `stellar-wallets-kit` | https://github.com/Creit-Tech/Stellar-Wallets-Kit |
| `tether-wdk` | https://github.com/tetherto/wdk |
| `tether-wdk-docs` | https://github.com/tetherto/wdk-docs |
| `phantom-connect-sdk` | https://github.com/phantom/phantom-connect-sdk |

## Soroban DeFi protocols

| Short name | Upstream |
|---|---|
| `blend-contracts` | https://github.com/blend-capital/blend-contracts (v2 at https://github.com/blend-capital/blend-contracts-v2, not cloned) |
| `defindex` | https://github.com/paltalabs/defindex |

## Agentic-payment protocols

| Short name | Upstream |
|---|---|
| `mpp-specs` | https://github.com/tempoxyz/mpp-specs (core HTTP-Payment-Auth draft file `draft-httpauth-payment-00.md` — submitted to IETF Datatracker as individual submission `draft-ryan-httpauth-payment`, currently `-01`; Stellar method draft `draft-stellar-charge-00`; specs CC0, tooling Apache-2.0 OR MIT) |
| `stellar-mpp-sdk` | https://github.com/stellar/stellar-mpp-sdk (source repo for the `@stellar/mpp` npm package; MIT) |
| `@stellar/mpp` | https://www.npmjs.com/package/@stellar/mpp (npm-published SDK; MIT; v0.5.0 as of 2026-04-15) |
| `mpp.dev` | https://mpp.dev (spec landing page; co-developed by Tempo and Stripe) |

## Other references cited in research without local clones

Several research files cite projects that were read only via the public web at the time of research, without a local clone. Cited URLs are inline in the relevant research file.

- MoonPay Open Wallet Standard — https://github.com/open-wallet-standard
- Trust Wallet Agent Kit (`twak`) — the source monorepo `github.com/trustwallet/twak` is private; the npm-published CLI bundle `@trustwallet/cli` is MIT and available via npm.
- Coinbase Agentic Wallet / CDP Wallets, Turnkey, Privy, Fireblocks, Safe + modules, MetaMask Snaps, LaunchTube, OpenZeppelin Relayer + Channels Plugin — public documentation URLs cited inline in the respective research files.
- `gh`, `stripe`, `aws` CLIs — public docs and repositories cited inline in `research/external/non-crypto/`.

## Notes on citation format

Research files cite upstream material in two ways:

- **Path + line anchor.** `stellar-cli/cmd/crates/soroban-cli/src/commands/.../mod.rs:NN` references a path relative to the repo root plus a line number. Readers should verify at the commit hash recorded in the research file's preamble, as `main` drifts.
- **URL with section anchor.** Public documentation (SEPs, CAPs, SDF blog posts, OpenZeppelin release notes) is cited by URL with a section-heading anchor where available.

When a cited path cannot be resolved against the upstream's current `main`, the recourse is to fetch the recorded commit hash. If no commit hash is recorded, treat the citation as frozen at the date in the research file's preamble (typically 2026-04 for the first wave of research).
