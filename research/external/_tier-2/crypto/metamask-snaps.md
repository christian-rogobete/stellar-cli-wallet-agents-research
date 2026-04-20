# MetaMask Snaps

**Status:** complete
**Researcher:** research agent (tier-2 wave)
**Last updated:** 2026-04-18
**ID:** metamask-snaps
**Category:** crypto-agent

---

## 1. What it is

MetaMask Snaps is the extension system of the MetaMask browser wallet. A Snap is a third-party JavaScript program that MetaMask loads into a sandboxed execution environment and wires into a defined set of wallet hooks. Snaps add chains, inject transaction-insight UI, run cron tasks, manage custom accounts, resolve names, or expose custom JSON-RPC endpoints to dapps and to other Snaps. Snaps have no inherent power; every capability is gated by a declarative permission in `snap.manifest.json` that the user approves at install time. This is the closest public prior art for a vetted-plugin boundary inside a self-custodial wallet and maps directly to `research/brain-dump/requirements.md` bullet 6 ("skills + extensibility") [S1] [S2].

## 2. Agent interface

Not an agent surface. Snaps are invoked by dapps or by MetaMask internals over a postMessage RPC; the user-facing trigger is a browser extension UI. The closest analogue to an "agent call" is a dapp's `wallet_requestSnaps` followed by `wallet_invokeSnap`, which routes to the Snap's `onRpcRequest` handler [S6]. The install flow uses `npm:<packageName>` as the canonical Snap ID [S4]. Relevant to our project purely as an extensibility-model reference, not as a runtime we would deploy against.

## 3. Custody model

Out-of-band: the MetaMask extension itself holds the root seed, and keys never leave the extension. Snaps may *request* scoped key material via `snap_getBip32Entropy` / `snap_getBip44Entropy` / `snap_getEntropy` (derived from the extension seed) or may register a custom account backend via `endowment:keyring` / `snap_manageAccounts` [S2] [S3]. These five methods are flagged as "account management permissions" and are the only ones that trigger a mandatory third-party audit before allowlisting [S3].

## 4. Policy and approval model

Install-time consent only, by default. The user sees the manifest's `initialPermissions` at install and must approve them before the Snap ever runs [S8]. There is no built-in per-invocation gate and no amount / counterparty / rate-limit layer; any transaction-level policy is the responsibility of the host wallet's signing confirmation, of the Snap itself (via `endowment:transaction-insight` / `endowment:signature-insight` which produce pre-sign UI [S6]), or of `allowedOrigins` caveats on `endowment:rpc` [S9]. Policy does not survive a compromise of the Snap once installed — it is bounded only by the permissions the user originally accepted.

## 5. Delegation model

No native delegation. A Snap runs in its own SES Compartment and its storage is private to that Snap [S7]. Authority flows outward only: the host extension delegates capabilities to the Snap via manifest permissions; the Snap does not delegate to sub-snaps. Snaps can expose RPC to other Snaps, but the boundary is a request/response call, not a revocable credential. Revocation is "uninstall the Snap."

## 6. Relevant protocol coverage

Snap permission taxonomy (manifest `initialPermissions`):

- **Snap-API RPC methods** — e.g. `snap_dialog` (modal UI), `snap_notify`, `snap_manageState` (persistent storage), `snap_manageAccounts` (custom account backend), `snap_getBip32Entropy` / `snap_getBip32PublicKey` / `snap_getBip44Entropy` / `snap_getEntropy` (scoped key material) [S2] [S3].
- **Endowments** — `endowment:cronjob` (scheduled background runs), `endowment:ethereum-provider` (MetaMask EIP-1193 provider), `endowment:page-home` (dedicated home UI surface), `endowment:keyring` (custom-account handler), `endowment:lifecycle-hooks` (`onInstall` / `onUpdate`), `endowment:name-lookup` (address/domain resolution), `endowment:network-access` (global `fetch`), `endowment:rpc` (dapp-or-snap-facing RPC; supports `allowedOrigins` caveat), `endowment:signature-insight` (pre-approval signature inspection), `endowment:transaction-insight` (pre-approval tx inspection), `endowment:webassembly` (WASM) [S2].
- **Dynamic / deprecated** — `eth_accounts` (requested at runtime); `endowment:long-running` (deprecated on MetaMask stable, still allowed in Flask) [S9].

Most endowments accept a `maxRequestTime` caveat bounded between 5000 ms and 180000 ms, defaulting to 60000 ms [S2]; notably `endowment:network-access` and `endowment:ethereum-provider` have no stated time bound [S2].

Entry points / hooks: `onRpcRequest`, `onTransaction`, `onSignature`, `onHomePage`, `onInstall`, `onUpdate`, `onCronjob`, `onNameLookup`, `onKeyringRequest`, `onUserInput` [S6].

## 7. Licence and adoption signals

- Licence: **proprietary** (Consensys custom licence: non-commercial, charitable/educational, or <10k MAU; commercial use requires a separate agreement) [S10].
- Source available: yes (read-only), at `github.com/MetaMask/snaps`.
- Last meaningful commit: active; 236 releases, latest `152.0.0` on 2026-04-15 [S11].
- GitHub stars: 828 [S11].
- Known production users: Snaps listed in the official directory (`snaps.metamask.io`) include Solana, Bitcoin, Cosmos, Starknet, Aptos, Sui, and many more chains integrated via Snaps; third-party audit firms (Cure53, Consensys Diligence, OtterSec, Sayfer, Hacken) run an approved-auditor programme [S5] [S12] [S13].
- Commercial backing: Consensys.
- Ecosystem traction: hundreds of Snaps audited and allowlisted since the September-2023 stable launch.

## 8. Non-negotiable check

Against `analysis/00-context.md` §4. Snaps is an extension *model*, not a standalone custody product; marks below refer to what a wallet built around the Snaps pattern inherits.

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | MetaMask itself is self-custodial; Snaps receive only derived material via `snap_getBipXX` [S2] [S3]. |
| N2 | Autonomous (no project-operated backend) | no | The allowlist is operated by the MetaMask Snaps team; installs on MetaMask stable are restricted to allowlisted Snaps by package name + version + shasum [S3] [S13]. |
| N3 | No central server for keys/policy/history | partial | Keys stay local. The allowlist-distribution channel is centralised; the permission-enforcement layer is local. |
| N4 | Open source, permissive licence | **no** | Consensys custom licence is non-commercial / capped-MAU; not permissive [S10]. |
| N5 | JSON-default output | n/a | Browser-extension UI model. |
| N6 | Testnet/mainnet parity | n/a | Inherits from the host wallet, not from Snaps. |

Snaps fails N2 and N4 as a deployment model. It remains a *design* reference for the extensibility boundary — which is the only purpose it serves here.

## 9. What to adopt (skills / extensibility model for our wallet)

Design proposal, not wholesale adoption. Each item cites the actor it serves and the threat it addresses.

- **Declarative permission manifest at install time** — a `skill.manifest.toml` (or JSON) with an `initialPermissions` section the wallet parses before load; direct analogue of Snap's `initialPermissions` [S8]. Addresses **T8**; serves **A1**, **A2**.
- **Narrow, typed permission taxonomy with named capabilities, not free-form grants** — our equivalent set, drawn from Snap endowments [S2]: `wallet:read-balance`, `wallet:read-tx-history`, `wallet:simulate-tx`, `wallet:sign-payment` (with amount + asset + counterparty caveats), `wallet:sign-soroban-auth-entry`, `wallet:derive-subaccount` (with SEP-5 path prefix), `net:fetch` (with `allowedOrigins` caveat), `net:horizon-rpc`, `ui:dialog`, `ui:home-page`, `storage:skill-private`, `schedule:cron` (with `maxRequestTime` equivalent), `insight:pre-sign-tx`, `insight:pre-sign-auth-entry`. The rule is: if there is no caveat-able noun, the capability does not exist. Addresses **T2**, **T4**, **T8**.
- **SES-style hardened-JavaScript sandbox per skill, one Compartment per skill, private storage** — mirror Snap's isolation boundary [S7]. Skills do not share state; skill-A cannot read skill-B's storage; skills have no DOM, no Node.js built-ins without a polyfill opt-in, no timers with sub-millisecond precision (side-channel guard) [S14]. Addresses **T1**, **T8**.
- **Install dialog shows the full permission set, wallet-owned UI** — no skill can render its own install prompt. Mirrors the MetaMask install flow [S8]. Addresses **T9**.
- **Per-capability caveats on install + first-invoke gate for policy-sensitive capabilities** — go one step beyond Snap: `wallet:sign-payment` additionally fires a wallet-owned approval on the *first call with novel parameters* (new destination, new asset, novel amount bucket). This closes the gap Snap leaves between install and runtime behaviour.
- **Mandatory audit for skills that request key-material or signing capabilities** — mirror Snap's hard rule: the five account-management methods trigger third-party audit before allowlisting [S3]. Our equivalent: any skill requesting `wallet:sign-*`, `wallet:derive-subaccount`, or `wallet:delegate-policy` must ship an audit commit-hash reference, verified at install.
- **On-chain / registry-gated WASM-hash attestation for signing-capability skills** — extend Snap's shasum + allowlist [S3] by publishing a signed attestation registry keyed on skill hash (sub-linear to the OZ attestation-registry pattern cited in `research/external/_summary.md`). Addresses **T8**.
- **Skill distribution as signed OCI/npm artifacts with package-name + version + shasum pinning**, mirroring MetaMask stable's rule that installs are restricted by `(package, version, shasum)` [S13]. Skills cannot be installed from a file path on mainnet profiles. Addresses **T8**.
- **Per-skill audit log (append-only JSONL)** — every permission-gated call the skill makes is logged with timestamp, capability name, argument keys (not values), and result category; exposed to the human operator (**A7**). No equivalent in Snap natively; closes an accountability gap the Consensys Diligence Shapeshift report identified [S15].

## 10. What to avoid (Snap-specific anti-patterns)

- **No hard per-capability time limit on long-running or cron-like skills.** Snap's original `endowment:long-running` had no time cap and was deprecated on stable only after repeated audit findings that RPC handlers could run indefinitely [S9] [S15]. Our equivalent must ship with a mandatory, non-optional `maxRequestTime` and cron-interval floor from day one.
- **Install-time consent as the only gate.** Snap exposes no first-invoke gate per capability; once `endowment:network-access` is granted the Snap can call `fetch` to any origin without a re-consent. For signing capabilities this would fail **T2**. Require `allowedOrigins`-style caveats as mandatory, not optional, on every network and signing capability.
- **Timers with full precision and shared global state in the sandbox.** MetaMask Snaps documentation explicitly warns about side-channel risks from `Date.now` precision and sensitive data in console logs [S14]. Design the sandbox with reduced-precision timers and a stripped console by default.
- **Argument validation asserted *before* canonicalisation.** The OtterSec property-spoofing bypass showed that a malicious Snap could attach a `toJSON()` to an RPC argument, pass the pre-sanitisation assertion, then mutate the call during `JSON.stringify` to bypass the permission system and invoke `eth_sendTransaction` without `endowment:ethereum-provider` [S16]. Our wallet must canonicalise first, then assert, and freeze the canonicalised object before dispatch. Addresses **T2**, **T8**.
- **Plaintext `console` logging of sensitive fields in skills.** The security guidelines instruct developers to remove logs, but the sandbox does not enforce it [S14]. Our runtime must redact argument values from any console output by default.
- **`endowment:network-access` without allowlisted origins as the default.** The Consensys Diligence report on the Shapeshift Snap rated `endowment:network-access` a major finding when requested unnecessarily [S15]. Our equivalent must refuse to grant `net:fetch` without an explicit `allowedOrigins` list.
- **Proprietary licence on the reference implementation.** Consensys's custom licence on the Snaps monorepo [S10] fails our **N4**. If we take code patterns we re-implement under Apache-2.0 / MIT; no copy-paste.
- **Centralised allowlist as the only install gate.** MetaMask itself is moving off this with the "permissionless" directory experiment [S12]; we should ship the signed-attestation registry from day one and not build a soft-central gate we later have to unwind.

## 11. Open questions

- Per-skill resource caps (CPU, memory, WASM instance count) — Snap documents `maxRequestTime` but not memory; does our runtime need hard ceilings per capability?
- Cross-skill RPC (a skill calling another skill): Snap allows it through `endowment:rpc` with `snaps: true`; is that a capability we want from day one, or a future extension?
- Update model: MetaMask re-approves permission diffs on update; how do we handle a skill that wants a strictly-expanded permission in a new version?
- Skill-initiated approval UX: Snap's `snap_dialog` lets the skill render inside a MetaMask-owned modal. What is the Stellar-AI-wallet equivalent, and how do we prevent control-character / markdown injection into approval dialogs (Consensys Diligence finding on Shapeshift [S15])?
- Exact audit-trigger list for our capability set — which specific capabilities should be the hard-gate-plus-audit set (Snap's is 5 methods).

## 12. Sources

All sources accessed 2026-04-18.

- **[S1]** `https://docs.metamask.io/snaps/` — Snaps docs home.
- **[S2]** `https://docs.metamask.io/snaps/reference/permissions/` — full permission reference, endowment list, `maxRequestTime` bounds.
- **[S3]** `https://docs.metamask.io/snaps/how-to/get-allowlisted/` — allowlist requirements, audit-triggering methods, two-approver rule.
- **[S4]** `https://docs.metamask.io/snaps/how-to/publish-a-snap/` — npm distribution, `npm:<packageName>` install ID, allowlist step.
- **[S5]** `https://support.metamask.io/configure/snaps/metamask-snaps-faq/` — protected vs. open permissions, audit requirement for account-management permissions.
- **[S6]** `https://docs.metamask.io/snaps/reference/entry-points/` — `onRpcRequest`, `onTransaction`, `onSignature`, `onHomePage`, lifecycle hooks.
- **[S7]** `https://docs.metamask.io/snaps/learn/about-snaps/execution-environment/` — SES (Hardened JavaScript Compartments), available globals, DOM/Node absence.
- **[S8]** `https://docs.metamask.io/snaps/how-to/request-permissions/` — `initialPermissions` manifest syntax; install-time consent model.
- **[S9]** `https://docs.metamask.io/snaps/learn/best-practices/security-guidelines/` — `endowment:long-running` deprecation note, `allowedOrigins`, minimum-permissions principle.
- **[S10]** `https://github.com/MetaMask/snaps/blob/main/LICENSE` — Consensys custom non-commercial / <10k-MAU licence.
- **[S11]** `https://github.com/MetaMask/snaps` — 828 stars, 236 releases, latest `152.0.0` 2026-04-15.
- **[S12]** `https://permissionless.snaps.metamask.io/` — MetaMask's experimental permissionless directory, stated move away from centralised gatekeeping.
- **[S13]** `https://github.com/MetaMask/snaps/discussions/1411` — open-beta readiness guide: stable installs restricted to `(npm package name, version, shasum)`; requirement for public source and third-party audit for allowlisting.
- **[S14]** `https://docs.metamask.io/snaps/learn/best-practices/security-guidelines/` — side-channel warnings (`Date.now`), logging, SES compatibility, key-exposure guidance (same URL as S9, cited separately for distinct content).
- **[S15]** `https://diligence.consensys.io/audits/2023/07/metamask/partner-snaps-shapeshift-snap/` — Consensys Diligence Shapeshift Snap audit: `endowment:long-running` unbounded execution, superfluous `endowment:network-access`, control-character injection into `snap_dialog`, exposed accounts without consent.
- **[S16]** `https://osec.io/blog/2023-11-01-metamask-snaps/` — OtterSec "Playing in the Sand": iframe + LavaMoat + SES three-layer sandbox description; `toJSON`-based permission-bypass exploit in RPC argument validation; fix by post-canonicalisation assertion.

Snaps is tier-2; research was conducted against public documentation and published source only, not a local clone.

## 13. Cross-links

- Primary cross-link: `research/brain-dump/requirements.md` bullet 6 (skills + extensibility) — Snaps is our primary reference for this requirement, previously uncited in tier-1 (see `research/external/_summary.md`).
- Threat-model cross-links: `analysis/02-threat-model.md` **T8** (supply-chain compromise of plugins — Snaps is the reference point for attestation + sandbox design), **T2** (prompt-injected transaction — informs the first-invoke-gate pattern), **T9** (approval-channel manipulation — informs the wallet-owned install dialog).
- Actor cross-links: `analysis/01-actor-model.md` **A1**, **A2** (skill extensibility is needed for unattended / orchestrator actors), **A7** (per-skill audit log).
- Compares to: `safe.md` §9 on-chain module-attestation registry (ERC-7484/7512); Safe is the on-chain policy-module analogue, Snaps is the off-chain sandbox-plugin analogue. The two together inform our skills model: Snap-style sandbox boundary + Safe-style attestation registry.
- Compares to: `gh-cli.md` §10 and `stripe-cli.md` §10 — both ship plugin systems *without* signature verification or sandboxing. Snaps is the counter-example we point to when arguing our wallet must do better than those CLIs.
