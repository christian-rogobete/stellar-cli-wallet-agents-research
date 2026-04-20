# Freighter + Stellar Wallets Kit

**Status:** complete
**Researcher:** research-agent (Opus 4.7, 1M)
**Last updated:** 2026-04-18
**ID:** freighter-walletkit
**Category:** crypto-stellar

---

## 1. What it is

Freighter is SDF's reference non-custodial browser-extension wallet for Stellar. Keys live in extension storage; every user-visible approval is a wallet-owned popup ([`freighter/README.md:1-6`](https://github.com/stellar/freighter/blob/master/README.md:1-6) @ `a7050e1`). Stellar Wallets Kit (`Creit-Tech/Stellar-Wallets-Kit`) is the community connector that exposes one SEP-43-shaped API and dispatches to pluggable modules for Freighter, xBull, Albedo, Lobstr, Rabet, Hana, Hot, Klever, OneKey, Bitget, Ledger, Trezor, and WalletConnect ([`stellar-wallets-kit/README.md:55-68`](https://github.com/Creit-Tech/Stellar-Wallets-Kit/blob/main/README.md:55-68), `src/sdk/modules/utils.ts:20-34` @ `a5b29ec`).

## 2. Agent interface

Freighter-API is a browser-only TS package. A caller does `isConnected()` → `requestAccess()` → `signTransaction(xdr, { networkPassphrase, address })` → `{ signedTxXdr, signerAddress }` (`@stellar/freighter-api/src/index.ts:1-43`, `signTransaction.ts:6-31`). Internally each call `postMessage`s an `EXTERNAL_SERVICE_TYPES.SUBMIT_TRANSACTION` request to the content script (`@shared/constants/services.ts:75-88`, `@shared/api/external.ts:69-104`); the background handler parses the XDR, runs memo-required / Blockaid / flagged-keys checks, opens `/sign-transaction?<encodedBlob>`, and resolves only on approval (`extension/src/background/messageListener/freighterApiMessageListener.ts:348-511`). `signAuthEntry` and `signMessage` follow the same three-phase pattern over `SUBMIT_AUTH_ENTRY` / `SUBMIT_BLOB` (`signAuthEntry.ts:10-47`, `signMessage.ts:26-60`). All methods return `FreighterApiNodeError` outside a browser (`signTransaction.ts:30`).

Wallets Kit wraps that shape: `StellarWalletsKit.init({ modules: defaultModules() })`, `getAddress()`, `signTransaction(xdr, { networkPassphrase, address })` (`README.md:22-47`). Its `ModuleInterface` mirrors SEP-43 — `getAddress`, `signTransaction`, `signAuthEntry`, `signMessage`, `getNetwork`, plus optional `signAndSubmitTransaction` (`src/types/mod.ts:145-232`). The Freighter module is a thin adapter over `@stellar/freighter-api` (`src/sdk/modules/freighter.module.ts:79-101`); WalletConnect uses Reown AppKit on `stellar:pubnet`/`stellar:testnet` with methods `stellar_signXDR` / `stellar_signAndSubmitXDR` but does not implement `signAuthEntry` or `signMessage` (`wallet-connect.module.ts:100-205,248-285`).

## 3. Custody model

Single tier: encrypted on disk in the extension, unlocked in-memory on password, never leaves the device. Dapp/agent holds a channel, not a key. Wallets Kit adds no custody. For agents this means Freighter cannot serve an unattended host (no non-interactive sign path) — every signature needs a human popup.

## 4. Policy and approval model

Domain-grant plus per-transaction human approval. The extension stores an allowlist of origins (`REQUEST_ACCESS`) and still pops up for every `SUBMIT_TRANSACTION`. The popup shows Summary / Operation-Details / Raw-XDR with Blockaid risk, memo-required, non-SSL, and Soroban auth-chain highlights (`extension/src/popup/views/SignTransaction/index.tsx:45-80`, `docs/docs/guide/signXdr.md:14-21`). There are no amount caps, counterparty allowlists, time windows, or rate limits. Clears T9 (approval channel is wallet-owned); does not address T2, T4, T6.

## 5. Delegation model

None. No session-keys, capabilities, or scoped signatures in either codebase. Any delegation must sit one layer down in SEP-10 ephemeral keys or a Soroban smart-account policy — and even then Freighter sees each auth-entry one by one (`@stellar/freighter-api/src/signAuthEntry.ts`). T4 is not mitigated here.

## 6. Relevant protocol coverage

Mainnet, testnet, futurenet; Wallets Kit also enumerates sandbox/standalone (`src/types/mod.ts:5-12`). **SEP-43 "Standard Web Wallet API Interface"**, Draft v1.2.1 (`stellar-protocol/ecosystem/sep-0043.md`, accessed 2026-04-18), is the canonical shape both implementations follow. Soroban auth-entry signing of `HashIdPreimageSorobanAuthorization` is first-class. WalletConnect v2 namespace is `stellar:pubnet`/`stellar:testnet`.

## 7. Licence and adoption signals

- Licence: Freighter Apache-2.0 ([`freighter/LICENSE`](https://github.com/stellar/freighter/blob/master/LICENSE)); Wallets Kit MIT ([`stellar-wallets-kit/LICENSE`](https://github.com/Creit-Tech/Stellar-Wallets-Kit/blob/main/LICENSE)).
- Source available: yes, both.
- Last meaningful commit: Freighter `a7050e1` (2026-04); Wallets Kit `a5b29ec` (2026-04).
- Known production users: Freighter itself; Wallets Kit on `lab.stellar.org`, `swap.xbull.app`, `mainnet.blend.capital`, `app.fxdao.io`, `app.sorobandomains.org`, `stellar.cables.finance` (`README.md:75-82`).
- Backing: SDF (Freighter); Creit Technologies LLP (Wallets Kit).
- Roadmap: SDF 2025 roadmap lists Freighter mobile (Q3 2025), in-wallet "Discover" dApp browser (Q3 2025), reusable Freighter backend (Q4 2025), and "Advanced authentication … leverag[ing] Soroban's smart wallet capabilities … progressive security based on held value and user-friendly key recovery options like passkeys and social sign-ins" (Q4 2025) (`stellar.org/foundation/roadmap`, accessed 2026-04-18).

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | Keys on-device only. |
| N2 | Autonomous (no project-operated backend) | partial | Signing is local, but Freighter defaults to `freighter-backend-prd.stellar.org` for indexer/balance/memo-required ([`freighter/README.md:42-54`](https://github.com/stellar/freighter/blob/master/README.md:42-54)). Wallets Kit is client-side. |
| N3 | No central server for keys/policy/history | partial | Keys yes; policy absent; history via SDF backend by default. |
| N4 | Open source, permissive licence | yes | Apache-2.0 / MIT. |
| N5 | JSON-default output | yes | Typed JSON returns; no CLI. |
| N6 | Testnet/mainnet parity | yes | `networkPassphrase` required; PUBLIC/TESTNET/FUTURENET enumerated. |

Design reference and **interop target**, not a deployment model for our wallet.

## 9. What to adopt

Our wallet must interoperate with this stack because actor A4's human side already has Freighter or expects the Wallets Kit connect button. Integration points we commit to:

- **Implement SEP-43 on our companion surface** with the exact signatures in `sep-0043.md` and `src/types/mod.ts:145-210` so our wallet drops in as a new Wallets-Kit module with zero dapp-side change. (A4)
- **Publish a Wallets-Kit `ModuleInterface` module** for our wallet: `ModuleType`, `productId/Name/Url/Icon`, `isAvailable` <1000 ms, `isPlatformWrapper` (`src/types/mod.ts:78-145`). (A4)
- **Speak WalletConnect v2** on `stellar:pubnet`/`stellar:testnet` for mobile approval, at minimum `stellar_signXDR`, ideally `stellar_signAndSubmitXDR` (`wallet-connect.module.ts:160-205,275-285`). Fill the gap the Creit module leaves by also implementing `signAuthEntry` and `signMessage` over WalletConnect (`wallet-connect.module.ts:248-260`). (A4, T9)
- **Keep the approval channel wallet-owned** — popup rendered by the wallet, never HTML passed by the caller (`extension/src/popup/views/SignTransaction/index.tsx`). (T9)
- **Tri-pane review** (summary / operation-details / raw-XDR) copied from Freighter's `/sign-transaction` popup, augmented with our policy verdict (allow / deny-by-rule / needs-human). (T2, T3)
- **Typed request envelopes**: one symbol per call, schema-validated at the trust boundary, mirroring `EXTERNAL_SERVICE_TYPES` (`@shared/constants/services.ts:75-88`). (T3)
- **`signAuthEntry` is a peer of `signTransaction`**, not an afterthought — every Soroban chain requiring multi-signer authorisation must be expressible (`@stellar/freighter-api/src/signAuthEntry.ts`). (A2, A3)

## 10. What to avoid

- **Central indexer as baseline.** Freighter's `INDEXER_URL` defaults to SDF-operated (`README.md:42-54`). Fails N2/N3.
- **Human-gated-only approval.** No per-tx/per-period/per-counterparty caps anywhere in `extension/src/background/messageListener/`. Fine for A7, fails T2/T4/T6 for A1/A2/A3. We need a local policy engine; we do not inherit one.
- **Browser-only runtime.** `FreighterApiNodeError` outside browser (`signTransaction.ts:30`). Our wallet cannot be a browser extension.
- **Key-based delegation.** Neither project has a delegation primitive; account-linking is not it. (T4)
- **WalletConnect gaps.** Creit's WC module rejects `signAuthEntry`/`signMessage` (`wallet-connect.module.ts:248-260`). We must implement.
- **Pinning to Draft SEP.** SEP-43 is Draft v1.2.1; target a pinned commit and re-verify before mainnet.

## 11. Open questions

- Does Freighter's Q4-2025 "Advanced authentication" expose a *smart-account policy surface* via the external API, or internal UX only? Roadmap entry is one sentence.
- Will SDF converge on SEP-43 or propose a successor that handles policy/delegation natively?
- Is the reusable Freighter backend (Q4 2025) self-hostable?
- Will Wallets Kit add `signAndSubmitTransaction` for non-WalletConnect modules, and will Freighter implement it?

## 12. Sources

- [`freighter/`](https://github.com/stellar/freighter) @ `a7050e1bff322dbe47683aaf805bdc174d58c7fc`, read 2026-04-18. Upstream `https://github.com/stellar/freighter` (Apache-2.0).
  - `@stellar/freighter-api/src/{index,signTransaction,signAuthEntry,signMessage}.ts`
  - `@shared/api/external.ts`, `@shared/constants/services.ts`
  - `extension/src/background/messageListener/freighterApiMessageListener.ts`
  - `extension/src/popup/views/SignTransaction/index.tsx`
  - `docs/docs/guide/signXdr.md`, `docs/docs/guide/thirdPartyIntegration.md`
  - `README.md`, `LICENSE`
- [`stellar-wallets-kit/`](https://github.com/Creit-Tech/Stellar-Wallets-Kit) @ `a5b29ec66d0194dcb35877894ff8d82f4b6f93f5`, read 2026-04-18. Upstream `https://github.com/Creit-Tech/Stellar-Wallets-Kit` (MIT).
  - `src/types/mod.ts`
  - `src/sdk/modules/{freighter,wallet-connect,utils}.{module.,}ts`
  - `README.md`, `LICENSE`
- `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0043.md`, accessed 2026-04-18 — SEP-43 Draft v1.2.1.
- `https://stellar.org/foundation/roadmap?locale=en`, accessed 2026-04-18 — Freighter Q3/Q4 2025 roadmap items (mobile, Discover, backend, Advanced authentication).

## 13. Cross-links

- Compares to `stellar-mcp-server.md` (agent-side vs human-side Stellar integration).
- Compares to `meridian-pay.md` (contract-side vs signer-side smart-account path).
- Surfaces candidate requirements (to be IDed in `03-requirements.md`): *wallet MUST implement SEP-43 and publish a Stellar-Wallets-Kit module*; *wallet MUST support WalletConnect v2 on `stellar:pubnet`/`stellar:testnet` for asynchronous human approval (A4)*.
