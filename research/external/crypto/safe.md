# Safe (+ modules)

**Status:** complete
**Researcher:** research-agent (opus-4-7)
**Last updated:** 2026-04-18
**ID:** safe
**Category:** crypto-agent

---

## 1. What it is

Safe (formerly Gnosis Safe) is an EVM smart-account (account-abstraction) contract suite. The core contract is an n-of-m multisig; extensibility is provided by **Modules** (contracts that can call back into the Safe to execute transactions without a signer quorum) and **Guards** (contracts that gate every transaction with a pre/post hook). Relevant here is not the product surface but the *module architecture* as the most-deployed reference for smart-account extensibility [12.1, 12.2, 12.3].

## 2. Agent interface

Programmatic use is chain-native. A caller (EOA or contract) invokes a deployed module (e.g. the AllowanceModule); the module authenticates the caller against its internal rules and, if permitted, calls `execTransactionFromModule(to, value, data, operation)` on the Safe, which executes without the usual signer threshold [12.4, 12.6]. End-to-end minimum signed action for a budgeted agent: owners enable the module via `enableModule(module)` (threshold-signed); owners call `setAllowance(delegate, token, amount, resetTimeMin, ...)`; the delegate signs an EIP-712 transfer hash and calls `executeAllowanceTransfer(...)`; module internally calls `execTransactionFromModule` [12.5, 12.6].

## 3. Custody model

Keys live with Safe **owners** (EOAs, hardware wallets, contract signers). Threshold defines the quorum. Modules are a *second custody tier*: once enabled, a module can spend without owner signatures, bounded only by the module's own logic. Guards form a third tier — gate both owner and module transactions [12.1, 12.3]. There is no vendor-held key; Safe{Wallet} is an interface, not custody.

## 4. Policy and approval model

Policy is on-chain and module-specific. Examples: AllowanceModule enforces per-token, per-period, per-delegate caps with a rolling reset [12.5, 12.6]; Zodiac Roles Modifier scopes allowed targets, selectors, and parameter values per role, with rate/threshold limits [12.7, 12.8]. Policy survives host and vendor compromise because enforcement is in the contract. Changes require owner-threshold transactions. The owner set itself is the trust root; if it is compromised, policy follows.

## 5. Delegation model

Delegation is expressed as **module attachment + module-internal policy**, not key sharing. The module holds a scoped capability: the right to call `execTransactionFromModule` on one Safe. Revocation is a threshold-signed `disableModule(prevModule, module)` call [12.2]. Effect is immediate from the next block. Sub-delegation patterns use Zodiac Roles where a role is assigned to an address with selector-level scope [12.7].

## 6. Relevant protocol coverage

- EVM chains; no Stellar/Soroban port exists. The "Safe Maths module" hit on Soroban searches is unrelated arithmetic, not account abstraction [12.9].
- ERC-4337 (user ops) via the 4337 module [12.4].
- ERC-7579 modular-account standard via the Safe7579 Adapter (Safe + Rhinestone) [12.10, 12.11].
- ERC-7484 / ERC-7512 on-chain module attestation registry (Rhinestone Registry) [12.10, 12.12].

## 7. Licence and adoption signals

- Licence: **LGPL-3.0** (`safe-smart-account`) [12.13].
- Source available: yes.
- Last meaningful commit: v1.5.0 release 2025-07-03 [12.13].
- GitHub stars: ~2.1k on `safe-smart-account` [12.13].
- Known production users: treasuries across most major DAOs; Gnosis Pay routes every transaction through Zodiac Roles [12.14].
- Commercial backing: Safe Ecosystem Foundation / Safe{DAO}.
- Ecosystem traction: Zodiac module family (Guild), Rhinestone module registry with attested catalogue, 14+ audited ERC-7579 modules [12.10, 12.12].

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | yes | Owners hold keys; no third party signs [12.13]. |
| N2 | Autonomous | yes | Contract-native; no required backend. Safe{Wallet} UI is optional. |
| N3 | No central server | yes | State is on-chain. Off-chain tx service is optional convenience. |
| N4 | Open-source permissive | **partial** | LGPL-3.0 is open but copyleft; acceptable for contract code, requires care when linked [12.13]. |
| N5 | JSON output | n/a | Not a CLI; JSON-RPC boundary is EVM-standard. |
| N6 | Testnet/mainnet parity | yes | Identical contracts deploy on both. |

Not a Stellar candidate. Design reference for §9.

## 9. What to adopt

- **Module-attachment as the delegation primitive.** On Stellar/Soroban, adopt a first-class "module" concept on smart accounts parallel to Safe's `enableModule` / `disableModule` / `execTransactionFromModule` trio [12.2, 12.4]. The OpenZeppelin `stellar-contracts` policy trait provides the closest analogue; it should be extended so that a policy can *initiate* an execution (Safe module direction), not only *enforce* one (current direction) [local: `packages/accounts/src/policies/mod.rs:47-162 @ 3f81125`].
- **On-chain attestation registry for modules.** Mirror ERC-7484/7512 (Rhinestone Registry): modules' WASM hashes carry signed attestations from named auditors; the smart account checks the registry on `add_policy` and optionally per-enforcement [12.10, 12.12].
- **Guard analogue for pre/post hooks.** Adopt Safe's `checkTransaction` / `checkAfterExecution` / `checkModuleTransaction` triplet as an optional `Guard` trait on the smart account, distinct from `Policy`; it gates every auth context including those initiated by policies themselves [12.3].
- **Explicit module-lifecycle events.** `ModuleEnabled`, `ModuleDisabled`, `ExecutionFromModuleSuccess`, `ExecutionFromModuleFailure` — the OZ `context_rule_added` / `policy_added` events are close but do not yet emit on module-initiated execution [12.6].

## 10. What to avoid

- **LGPL copyleft** for our reference implementation — pick Apache-2.0/MIT to match Stellar ecosystem norms and N4.
- **`operation = DelegateCall` in `execTransactionFromModule`.** Safe's module API lets a module request a delegatecall, collapsing the module's bytecode into the account's storage context — classic foot-gun [12.4]. On Stellar, do *not* give policies write access to the account's own storage keyspace.
- **Silent guard failure = full DoS.** Safe guards can brick an account if buggy [12.3]. Any Stellar guard must be removable without the guard's own consent (e.g. via a separate timelocked recovery rule).
- **Trusted-but-unverified modules.** Safe warns that modules can execute arbitrary transactions; only add trusted ones [12.1]. Relying on prose warnings fails T2/T8. Registry-gated installation is the fix.

## 11. Explicit comparison: Safe modules vs OZ stellar-contracts accounts

Both sides cited at `@ 3f81125` (OZ stellar-contracts) and `@ v1.5.0` (Safe).

| Capability | Safe | OZ stellar-contracts | Parity requirement on Stellar |
|---|---|---|---|
| Attach extension | `enableModule(address)` [12.2] | `add_policy(context_rule_id, address, install_param)` [local: `smart_account/storage.rs:1110`] | Parity: yes, similar surface. |
| Detach extension | `disableModule(prev, module)` [12.2] | `remove_policy(context_rule_id, policy_id)` [local: `storage.rs:1175`] | Parity: yes. |
| Extension-initiated execution | `execTransactionFromModule` [12.4, 12.6] | **Missing.** Policies only `enforce()` during `__check_auth` and cannot initiate calls [local: `policies/mod.rs:79-85`] | Add a `ModuleExecute` trait + `execute_from_policy()` entry on the account. |
| Module registry / attestations | ERC-7484 Rhinestone Registry [12.12] | None. | New contract; hash-pinned attestations keyed on WASM hash. |
| Transaction guards | `setGuard(address)` with `checkTransaction` / `checkAfterExecution` [12.3] | None (policies are scoped to a single rule, not global). | New `Guard` trait invoked for every context. |
| Module guards (gate module-initiated calls) | `setModuleGuard` + `checkModuleTransaction` [12.3] | n/a (no module-initiated execution yet). | Follows from guard + module work. |
| Role-based permissions | Zodiac Roles Modifier (target + selector + param scope) [12.7, 12.8] | `ContextRule` with `CallContract(Address)` type (contract-level only) [local: `smart_account/mod.rs:14-24`] | Extend context types to selector+args-level scoping or build a Roles policy. |
| Spending-limit module | AllowanceModule (per-delegate, per-token, rolling reset) [12.5] | `spending_limit` policy (rolling window) [local: `policies/spending_limit.rs`] | Parity: yes in policy form, but lacks module-initiated submission. |

The central gap is direction of control: OZ policies *gate* the account's `__check_auth`; Safe modules *drive* the account. For agent delegation (A2, A4) where a sub-agent acts on a budget without constructing a full owner-signed transaction, the Safe direction is strictly more expressive.

## 12. Sources

1. [12.1] `https://docs.safe.global/advanced/smart-account-modules`, accessed 2026-04-18. Module concept, warnings.
2. [12.2] `https://docs.safe.global/reference-smart-account/modules/enableModule`, accessed 2026-04-18. `enableModule`, `disableModule`, `isModuleEnabled`, `getModulesPaginated`.
3. [12.3] `https://docs.safe.global/advanced/smart-account-guards`, accessed 2026-04-18. Guard interface and v1.3.0 introduction.
4. [12.4] `https://docs.safe.global/reference-smart-account/modules/execTransactionFromModule`, accessed 2026-04-18. Function signature and events.
5. [12.5] `https://docs.safe.global/home/ai-agent-quickstarts/agent-with-spending-limit`, accessed 2026-04-18. AllowanceModule delegate model.
6. [12.6] `https://docs.safe.global/advanced/smart-account-modules/smart-account-modules-tutorial`, accessed 2026-04-18. Module build pattern.
7. [12.7] `https://github.com/gnosisguild/zodiac-modifier-roles`, accessed 2026-04-18. Zodiac Roles Modifier source and interface.
8. [12.8] `https://docs.roles.gnosisguild.org/`, accessed 2026-04-18. Roles permission scoping model.
9. [12.9] `https://github.com/stellar/sorobounty-spectacular/discussions/13`, accessed 2026-04-18. Unrelated Soroban "safe maths" submission (confirms no Safe account-abstraction port).
10. [12.10] `https://docs.safe.global/advanced/erc-7579/7579-safe`, accessed 2026-04-18. Safe7579 Adapter + 14 audited modules.
11. [12.11] `https://ackee.xyz/blog/rhinestone-erc-7579-safe-adapter-audit-summary/`, accessed 2026-04-18. Ackee audit of Safe7579 (June–July 2024).
12. [12.12] `https://docs.rhinestone.wtf/module-registry`, accessed 2026-04-18. Module Registry / ERC-7484 attestations.
13. [12.13] `https://github.com/safe-global/safe-smart-account`, accessed 2026-04-18. LGPL-3.0, v1.5.0 (2025-07-03), Certora/Ackee formal verification, 2.1k stars.
14. [12.14] `https://gnosisguild.mirror.xyz/oQcy_c62huwNkFS0cMIxXwQzrfG0ESQax8EBc_tWwwk`, accessed 2026-04-18. Gnosis Pay routes every tx through Zodiac Roles.
15. [12.15] [`stellar-contracts/packages/accounts/README.md`](https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/README.md) @ `3f81125`. Policy lifecycle, signer model, ContextRule semantics.
16. [12.16] [`stellar-contracts/packages/accounts/src/policies/mod.rs:47-185`](https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/policies/mod.rs:47-185) @ `3f81125`. `Policy` trait (`enforce`, `install`, `uninstall`) — no initiate-execute capability.
17. [12.17] [`stellar-contracts/packages/accounts/src/smart_account/storage.rs:632,845,931,992,1110,1175`](https://github.com/OpenZeppelin/stellar-contracts/blob/main/packages/accounts/src/smart_account/storage.rs:632,845,931,992,1110,1175) @ `3f81125`. `add_context_rule`, `remove_context_rule`, `add_signer`, `remove_signer`, `add_policy`, `remove_policy`.

## 13. Cross-links

- Compares directly to: `research/external/crypto-stellar/openzeppelin-stellar-accounts.md` (OZ policy trait vs Safe module trait).
- Informs: `analysis/04-smart-accounts.md` (when written) — module-attachment pattern, registry, guards.
- Related: `research/external/crypto/turnkey.md`, `research/external/crypto/coinbase-agentic.md` — alternative delegation models off-chain.
