# Requirements

**Status:** stable — 2026-04-21. Vocabulary locked; 214 REQs (108 main + 106 addendum); full-format expansion of new addendum entries deferred (addendum §7).
**Depends on:** `01-actor-model.md`, `02-threat-model.md`, `research/external/**`, `research/stellar-capabilities/**`
**Loop partners:** both research streams

---

## 0. How this document works

### 0.1 Source tagging

Every requirement carries a `sources:` field. Valid values:

- `A1`..`A7` — actor IDs from `01-actor-model.md`.
- `T1`..`T10` — threat IDs from `02-threat-model.md`.
- `research/external/<path>#<anchor>` — a specific section in a candidate file.
- `research/stellar-capabilities/<path>#<anchor>` — a specific section in a capability file.
- `research/brain-dump/requirements.md:<line>` — a bullet from the seed brain-dump, with line number.
- `team-judgment` — used sparingly and explicitly. If >~10% of requirements end up with only this source, that is a signal to go back into research.

A requirement with no source is a **gap**, not a requirement. Gaps live in §0.3.

### 0.2 Classification

- **MUST** — the wallet is not fit for purpose without this. Failure ships a broken product.
- **SHOULD** — expected. Absence requires a named, recorded trade-off.
- **NICE** — desirable, defer-safe, may be post-v1.
- **OUT** — explicitly out of scope. Listed so the question does not reopen unnoticed.

### 0.3 Gaps queue

Open research tasks: draft requirements without a citeable source. Drain by finding or producing a source, then relocating into the appropriate section below. Fully drained gaps are logged in §0.3.1 for provenance.

| ID | Draft requirement | Why it needs a source | Target research area |
|---|---|---|---|
| G2 | Whether the wallet owns the x402 payment-requirements cache, or defers to the calling HTTP client. | Pattern is young; no candidate ships a local cache design. | Stream-2 + SDF x402-MCP follow-up |
| G3 (narrowed) | CAP-0073 arithmetic finalisation for the minimum-reserve guard after the Protocol 26 mainnet vote (2026-05-06). Per-capability arithmetic is folded into `REQ-sec-policy-minimum-reserve` via the addendum §1 merge log (stream-2 sources from `01-accounts.md`, `02-classic-ops.md`, `07-dex-amm.md`). | CAP-0073 may alter the final reserve model; the wallet-side arithmetic has to follow. | `research/stellar-capabilities/01-accounts.md` + CAP-0073 tracker |
| G7 (narrowed) | Sequence-pool behaviour under re-org and `txBadSeq` recovery. Fee-bump interaction is drained by `REQ-perf-fee-bump` (addendum §4). | The in-process sequence pool still needs documented re-org and seq-err recovery semantics. | `research/stellar-capabilities/08-infra-ops.md` |

### 0.3.1 Drain log

Gaps fully closed by requirements elsewhere in this file or in `03-requirements-addendum.md`.

| Gap ID | Original draft | Drained by | Date |
|---|---|---|---|
| G1 | SEP-10 ephemeral-key management on the wallet side. | `REQ-sep-sep10-ephemeral` | 2026-04-21 |
| G4 | Subaccount semantics — BIP-44 derivation vs. C-account-per-child. | `REQ-acct-derivation-sep05` + `REQ-acct-subaccount-isolation` (addendum §3 split) | 2026-04-21 |
| G5 | Signer-plugin interface (credential-process-shaped). | `REQ-acct-signer-plugin` (interface shape) + `REQ-acct-hardware-signer` (Ledger) + `REQ-acct-keyring-first` (OS-keyring); passkey / TEE backends land through the plugin | 2026-04-21 |
| G6 | Policy-mutation audit-chain format. | `REQ-sec-audit-log-signed` (hash-chained format chosen) | 2026-04-21 |

### 0.4 Candidates queue

Items surfaced by research (external or Stellar capability) that may become requirements. Each is a review task: promote, reject, or merge with an existing requirement.

| ID | Description | Surfaced from | Proposed classification | Review status |
|---|---|---|---|---|
| C1 | Module-attestation registry (ERC-7484 / Rhinestone shape) for signed skill WASM hashes. | `research/external/crypto/safe.md#9`, `research/external/_tier-2/crypto/metamask-snaps.md#9` | SHOULD (v1.x) | defer; depends on skills ecosystem emerging |
| C2 | On-chain "Admin-Quorum" governance plane for policy edits themselves. | `research/external/_tier-2/crypto/fireblocks.md#9` | NICE | review after smart-account design locks |
| C3 | 2-share device+server reconstitution as an opt-in hosted mode for A4. | `research/external/_tier-2/crypto/privy.md#9` | OUT (v1) / NICE (v2) | conflicts with N3 unless the server is user-operated |
| C4 | Root-quorum bypass / explicit unbrick semantics (Turnkey break-glass). | `research/external/crypto/turnkey.md#9` | SHOULD | promote once A7 recovery UX is designed |
| C5 | `stripe trigger`-style testnet scenario fixtures (invoice-paid, refund-succeeded analogues on Stellar). | `research/external/non-crypto/stripe-cli.md#9` | NICE | useful for developer acquisition, not v1 correctness |
| C6 | `stripe listen --forward-to` subprocess pattern for Horizon SSE / Soroban `getEvents`. | `research/external/non-crypto/stripe-cli.md#9` | SHOULD | promote when streaming tool shape is designed |
| C7 | Seven-sub-spec modular-profile declaration à la OWS. | `research/external/crypto/moonpay-agents-ows.md#9` | NICE | useful for interoperability narrative, low v1 cost |
| C8 | Delegation-as-TEE-re-provisioning for A2/A4. | `research/external/_tier-2/crypto/privy.md#9` | OUT | requires TEE dependency conflicting with N1 posture |
| C9 | `raw_sign` primitive for out-of-band payload signing (SEP-45 server-side, custom flows). | inferred from Privy, Fireblocks | NICE | reject unless a named caller demands it — policy cannot reason about raw bytes (T2) |
| C10 | Multi-endpoint RPC cross-check for high-value actions (T7 defence). | `02-threat-model.md#T7` | SHOULD | **resolved → `REQ-sec-rpc-crosscheck`** (addendum §4) |
| C11 | Router / multicall contract to bundle invocations under one approval. | `research/external/stellar-ecosystem/meridian-pay.md#9` | NICE | review after A4 approval flow designed |
| C12 | Dedicated testnet marker printed on every response (not just network name). | `research/external/crypto/kraken-cli.md#9` ("paper mode" parallel) | SHOULD | merge into REQ-ux-network-marker design once UX locks |

### 0.5 Requirement entry format

```markdown
#### REQ-<area>-<slug>: <title>

- **Statement:** one atomic, testable sentence.
- **Classification:** MUST | SHOULD | NICE | OUT
- **Actors:** A#, A# (optional if covered by threat/capability sources)
- **Sources:** as per §0.1
- **Rationale:** 1-3 sentences, only where the statement is not self-justifying.
- **Acceptance signal:** how an implementer would verify this in a reference implementation.
```

One requirement per heading. Atomic: no compound statements joined by "and." If a bullet from `research/brain-dump/requirements.md` encodes several things, it gets split.

---

## 1. Functional

### 1.1 Account and key management

#### REQ-acct-key-isolation: Keys live inside the wallet trust boundary

- **Statement:** Signing-key material is never exposed to the agent runtime or to any caller outside the wallet core process.
- **Classification:** MUST
- **Actors:** A1, A2, A3, A4, A5, A7
- **Sources:** T1, `research/brain-dump/requirements.md:3`, `research/external/stellar-ecosystem/stellar-mcp-server.md#10`, `research/external/crypto/kraken-cli.md#9`
- **Rationale:** The agent-as-hostile frame collapses if the agent process can read the key. The existing Stellar MCP's `secretKey: z.string()` tool argument is the anti-pattern the non-negotiable rejects.
- **Acceptance signal:** Static check that no public tool schema accepts a secret-seed field; dynamic check that keys are zeroised from memory after each sign and never serialised to stdout/stderr.

#### REQ-acct-keyring-first: OS keyring is the default secret store

- **Statement:** On platforms with an OS keyring (macOS Keychain, Linux Secret Service, Windows DPAPI), the wallet stores signing secrets there by default.
- **Classification:** MUST
- **Actors:** A1, A5, A7
- **Sources:** T1, `research/external/non-crypto/gh-cli.md#9`, `research/external/stellar-ecosystem/stellar-cli.md#9`
- **Rationale:** Default posture decides most installations' actual security. `gh` and `stellar-cli` both demonstrate this pattern; plaintext-on-disk is the disqualifying anti-pattern across Stellar MCP and default `stellar-cli keys generate`.
- **Acceptance signal:** First-run setup writes to the OS keyring without prompts; `--insecure-storage` flag exists but emits a warning.

#### REQ-acct-insecure-storage-optout: Plaintext storage only under a named opt-out

- **Statement:** The wallet refuses to store secrets in plaintext unless the user passes an explicit flag named so it appears in shell history and in logs.
- **Classification:** SHOULD
- **Actors:** A5, A7
- **Sources:** T1, `research/external/non-crypto/gh-cli.md#9`, `research/external/stellar-ecosystem/stellar-cli.md#10`
- **Rationale:** `gh`'s `--insecure-storage` naming is load-bearing — users and reviewers see the risk in their history.
- **Acceptance signal:** Attempting keygen on a system with no keyring available and no `--insecure-storage` flag exits with a dedicated error code; flag name contains the word "insecure."

#### REQ-acct-hardware-signer: Hardware-wallet signing is a first-class signer backend

- **Statement:** The wallet can sign transactions and Soroban auth entries using a Ledger device without the secret material leaving the device.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** T1, `research/external/stellar-ecosystem/stellar-cli.md#9`, A7 ("Must-have capabilities")
- **Rationale:** A7 is the fastest-adoption actor and routinely runs in an IDE agent; hardware isolation is the single largest T1 reduction available on a dev laptop.
- **Acceptance signal:** Sign a classic payment and a Soroban invoke-contract auth entry from a connected Ledger; no keyring entry created for that identity.

#### REQ-acct-signer-plugin: External-signer plugin interface

- **Statement:** The wallet accepts signing through a named external process that receives a canonical signing request on stdin and returns a signature on stdout.
- **Classification:** SHOULD
- **Actors:** A5, A7
- **Sources:** `research/external/non-crypto/aws-cli.md#9` (`credential_process`), `research/external/stellar-ecosystem/stellar-cli.md#9` (Signer enum), T1
- **Rationale:** A single extension point absorbs passkey, TEE, HSM, and future signer kinds without modifying the core.
- **Acceptance signal:** A reference plugin that prints a pre-made signature is invoked on a named identity, the signature is accepted, and the core never touches the key material.

#### REQ-acct-multi-identity: Named identity profiles

- **Statement:** The wallet supports multiple named identities per install, each with its own keyring entry and address.
- **Classification:** SHOULD
- **Actors:** A1, A2, A5, A7
- **Sources:** `research/external/non-crypto/stripe-cli.md#9` (named profiles), `research/external/non-crypto/aws-cli.md#9` (two-layer config), `research/external/stellar-ecosystem/stellar-cli.md#9`
- **Rationale:** Same developer uses different keys for developing, staging, and mainnet operations; A5 often has a distinct deploy key per repo.
- **Acceptance signal:** Create two identities, select one via flag and one via env var; addresses differ; keyring entries are distinct.

#### REQ-acct-subaccounts: Deterministic subaccount derivation from a mnemonic

- **Statement:** The wallet derives subaccount keys from a single BIP-39 mnemonic using SLIP-10 ed25519 with a documented path.
- **Classification:** MUST
- **Actors:** A2, A5
- **Sources:** `research/brain-dump/requirements.md:23`, A2 ("Must-have capabilities"), `research/external/_tier-2/crypto/tether-wdk.md#10` (anti-pattern: treating same-seed BIP-44 as delegation)
- **Rationale:** A2 needs a scalable supply of addresses; deterministic derivation survives device loss given the mnemonic. C-account-per-child delegation is covered separately by `REQ-acct-subaccount-isolation` (addendum §3); G4 drained (§0.3.1).
- **Acceptance signal:** Derive address at index `i` reproducibly from the same mnemonic across reinstalls; documented path string present in output.

#### REQ-acct-address-export: Expose account address as a first-class read

- **Statement:** The wallet exposes the current identity's G-address through a dedicated command that succeeds with the keystore locked.
- **Classification:** MUST
- **Actors:** A4, A6, A7
- **Sources:** `research/brain-dump/requirements.md:10`, A6 ("clean separation of read operations")
- **Acceptance signal:** `stellar-ai wallet address` returns the G-address while the keyring is locked, exits 0, prints no secret material.

#### REQ-acct-mnemonic-import: Import from existing mnemonic

- **Statement:** The wallet supports importing an existing BIP-39 mnemonic into a named identity at first-run and later.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** `research/brain-dump/requirements.md:23`, team-judgment (recovery posture)
- **Rationale:** Users arriving from another Stellar wallet must be able to continue without key generation.
- **Acceptance signal:** Import a known mnemonic; derived address matches the expected G-address from a reference implementation.

### 1.2 Classic on-chain operations

#### REQ-classic-payment: Classic asset payments

- **Statement:** The wallet can construct, sign, and submit a classic payment operation for XLM and for any issued asset.
- **Classification:** MUST
- **Actors:** A1, A3, A4, A7
- **Sources:** `research/brain-dump/requirements.md:4`, A1 ("frequent, small, bounded")
- **Acceptance signal:** Send 1 XLM between two testnet accounts; send 1 unit of a testnet-issued asset; both transactions succeed and the wallet emits an ingestible receipt.

#### REQ-classic-balances: Read account balances

- **Statement:** The wallet returns account balances as a machine-readable structure per asset.
- **Classification:** MUST
- **Actors:** A1, A4, A6, A7
- **Sources:** `research/brain-dump/requirements.md:9`, A6
- **Acceptance signal:** Balance query returns XLM and every issued asset held, each with asset code, issuer, balance, and unit annotation; succeeds without a signing key configured.

#### REQ-classic-trustline: Trustline create and remove

- **Statement:** The wallet can create and remove trustlines for issued assets.
- **Classification:** SHOULD
- **Actors:** A1, A3, A7
- **Sources:** team-judgment (prerequisite for A3 payments in non-XLM assets), A3
- **Acceptance signal:** Create a trustline to a testnet issuer, receive payment, remove the trustline when the balance is zero.

#### REQ-classic-fee-estimate: Explicit fee estimation and fee-bump support

- **Statement:** The wallet returns an explicit fee estimate for every constructed transaction and supports fee-bump transactions.
- **Classification:** SHOULD
- **Actors:** A1, A5
- **Sources:** T3, A5 ("idempotent operations with explicit sequence-number handling"), `research/external/_tier-2/stellar-ecosystem/launchtube.md#9`
- **Rationale:** A1 and A5 both operate unattended and must budget for network congestion; surfacing the estimate separately from the submit is what makes policy evaluation possible.
- **Acceptance signal:** The pre-submission / build step returns fee as a first-class field; `submit` uses the same fee or a fee-bump envelope; re-run after network congestion shift produces a different estimate.

#### REQ-classic-memo: Display and require memo when receiver demands it

- **Statement:** The wallet refuses to send a payment to a memo-required destination without an explicit memo field on the request.
- **Classification:** MUST
- **Actors:** A1, A3, A4, A7
- **Sources:** T5, `research/external/stellar-ecosystem/freighter-walletkit.md#9`
- **Rationale:** Memo-required exchanges are the single most common user-loss pattern on Stellar.
- **Acceptance signal:** Sending to a memo-required destination without a memo fails at the wallet layer before signing and returns a typed error the agent can route on.

#### REQ-classic-path-payment: Path payments

- **Statement:** The wallet can construct and submit `pathPaymentStrictSend` and `pathPaymentStrictReceive` operations.
- **Classification:** SHOULD
- **Actors:** A1, A3
- **Sources:** `research/brain-dump/requirements.md:7` (trading), team-judgment
- **Acceptance signal:** Execute a path payment swapping two testnet assets; returned receipt carries the full path taken.

#### REQ-classic-sequence-pool: In-process sequence pool for A1-class workloads

- **Statement:** The wallet exposes an optional in-process deterministic-derivation sequence pool for frequent unattended submission.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** `research/external/_tier-2/stellar-ecosystem/launchtube.md#9`, `analysis/00-context.md#5.4`
- **Rationale:** LaunchTube is being discontinued and its self-hosted replacement is likely AGPL-3.0; shipping the primitive in-process both removes a dependency and positions the wallet cleanly against the SDF track 2 migration.
- **Acceptance signal:** Submit 50 transactions per minute from a single identity without `txBadSeq` errors using deterministic key derivation; documented re-org and `txBadSeq` recovery semantics (G7 narrowed; fee-bump covered by `REQ-perf-fee-bump`).

### 1.3 Soroban operations

#### REQ-sor-invoke: Invoke Soroban contract

- **Statement:** The wallet can simulate, authorise, and submit an `invokeHostFunction` for a named contract with typed scVal arguments.
- **Classification:** MUST
- **Actors:** A1, A2, A3, A7
- **Sources:** A7 ("deploy a Soroban contract"), `research/external/stellar-ecosystem/stellar-mcp-server.md#9`
- **Acceptance signal:** Invoke a testnet contract method with mixed scalar and composite arguments; receipt includes the returned scVal.

#### REQ-sor-simulate-first: Simulate is a first-class peer of submit

- **Statement:** Every Soroban-mutating tool has a paired simulate tool that returns the resolved transaction, auth entries, footprint, and resource fees without signing.
- **Classification:** MUST
- **Actors:** A1, A2, A3, A6, A7
- **Sources:** T3, `research/external/_tier-2/crypto/phantom-mcp-server.md#9` (simulate as peer, not flag), `research/external/stellar-ecosystem/stellar-mcp-server.md#9`
- **Rationale:** T3 requires previews before signature; making simulate a separate tool is what allows an agent runtime or policy to inspect the outcome without opportunistic signing.
- **Acceptance signal:** `simulate` and `send` exist as separate MCP tools; their JSON schemas return the same resolved-transaction shape.

#### REQ-sor-auth-entries: Sign Soroban auth entries out-of-band

- **Statement:** The wallet exposes `signAuthEntry` as a distinct operation that signs a single `SorobanAuthorizationEntry` without submitting a transaction.
- **Classification:** SHOULD
- **Actors:** A2, A3
- **Sources:** `research/external/stellar-ecosystem/freighter-walletkit.md#9` (SEP-43 + WC gaps), A3 (x402 flow), T4
- **Rationale:** x402, delegation to sub-agents, and multi-party Soroban transactions all depend on this primitive; Creit's Stellar Wallets Kit WalletConnect module rejects it as a named method (`freighter-walletkit.md` §9-10).
- **Acceptance signal:** Produce a valid `SorobanAuthorizationEntry` signature that is accepted by a separate submitter process; entry is scoped to the given invocation and nonce.

#### REQ-sor-deploy: Deploy Soroban contract

- **Statement:** The wallet can upload WASM and deploy a Soroban contract instance under a given identity.
- **Classification:** SHOULD
- **Actors:** A5, A7
- **Sources:** A7 ("draft a Soroban contract, deploy it to testnet"), `research/external/stellar-ecosystem/stellar-cli.md#9`
- **Acceptance signal:** Deploy a WASM file to testnet; returned receipt includes contract ID; subsequent invoke on that ID succeeds.

#### REQ-sor-simulation-audit: Simulation-auth matches submission-auth

- **Statement:** The wallet verifies that the auth entries produced by simulation are equivalent to the ones actually signed, and refuses to submit if they differ without an explicit override.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** `research/external/_tier-2/stellar-ecosystem/launchtube.md#9`, T3, T7
- **Rationale:** LaunchTube's simulation-auth-matches-submission-auth audit is the exact primitive that catches an RPC endpoint silently modifying footprint between simulate and submit.
- **Acceptance signal:** Tamper with the footprint between simulate and submit in a test harness; wallet rejects and emits a typed error.

#### REQ-sor-footprint-cap: Resource-fee and footprint caps in policy

- **Statement:** Policy rules can limit Soroban resource fees and footprint-entry counts.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** T6, `research/external/crypto/coinbase-agentic.md#9` (typed criteria)
- **Rationale:** Runaway loops in Soroban are denominated in resource fees, not just XLM, and must be caps-able at the same layer as value caps.
- **Acceptance signal:** A policy that caps resource fees rejects a contract invocation whose simulation returns fees above the cap.

### 1.4 Smart accounts and delegation

#### REQ-sa-c-account-support: Operate from a C-account as signer source

- **Statement:** The wallet can use a Soroban smart-account contract (C-account) as the source of an authorising signature, not only a G-account.
- **Classification:** MUST
- **Actors:** A2, A4
- **Sources:** `research/brain-dump/requirements.md:21`, A2, `research/external/stellar-ecosystem/meridian-pay.md#9`, `research/external/stellar-ecosystem/stellar-mcp-server.md#9` (signer-shape routing)
- **Rationale:** G-accounts cannot carry programmable policy; smart accounts are how the RFP's policy story gets on-chain enforcement.
- **Acceptance signal:** Sign a Soroban invocation where the source is a C-account with a `__check_auth` implementation; submission succeeds on testnet.

#### REQ-sa-policy-initiated-exec: Policy-initiated execution (custom wallet machinery atop OZ)

- **Statement:** For delegation patterns that require a policy to *originate* an invocation rather than only accept or reject one (recurring payments, top-ups, allowance refills), the wallet ships custom machinery on top of OZ context rules. OZ v0.7.1 policies run only inside `__check_auth` and cannot originate transactions; this requirement captures the additional layer the wallet builds itself (for example, a scheduler actor authorised by a dedicated context rule that periodically constructs and submits the bounded payment transaction). This is explicitly not an OZ-provided feature — `04-smart-accounts.md` §7 records it as a gap.
- **Classification:** SHOULD
- **Actors:** A2, A4
- **Sources:** `research/external/crypto/safe.md#9` (Safe's module-attachment pattern, which OZ does not replicate), `04-smart-accounts.md#7` (gap list), T4
- **Rationale:** Recurring payments, top-ups, and allowance refills are delegation patterns that the wallet-layer scheduler enables by wrapping OZ's accept-or-reject policies with a separate authorised origination path. Keeping this as a SHOULD (rather than MUST) recognises it as a wallet-specific extension, not an OZ primitive.
- **Acceptance signal:** A wallet-layer scheduler authorised by a scoped OZ context rule initiates a bounded transfer without additional human authorisation; an unbounded transfer attempted through the same path is rejected by the context rule's spending-limit policy.

#### REQ-sa-webauthn-signer: Passkey (WebAuthn) signer for C-accounts

- **Statement:** The wallet's C-account template supports WebAuthn (passkey) as a signer kind.
- **Classification:** SHOULD
- **Actors:** A4
- **Sources:** `research/external/stellar-ecosystem/smart-account-kit.md#9` (SAK + KMP `OZSmartAccountKit` WebAuthn verifier path, primary); `research/external/stellar-ecosystem/meridian-pay.md#9` (custom `__check_auth` WebAuthn baseline, minimalism reference)
- **Rationale:** A4 (user-facing assistant) is the actor most served by biometric approval. Primary references: SAK and KMP `OZSmartAccountKit`, both driving the OZ-contract secp256r1 WebAuthn verifier across browser (SAK) and Android / iOS / macOS / Web (KMP). Meridian Pay's ~100 LOC single-file account remains useful as a minimalism reference for what `__check_auth` can look like without OZ.
- **Acceptance signal:** A passkey on a user device signs a C-account invocation; the signature is verified on-chain without a G-account keypair in the flow.

#### REQ-sa-revocation-takes-effect-before-next-sig: Revocation is synchronous

- **Statement:** Revoking a sub-agent's delegated authority takes effect before the next signature from that sub-agent can be produced.
- **Classification:** MUST
- **Actors:** A2, A4, A7
- **Sources:** T4, A2 ("uncomfortable failure mode"), `research/external/crypto/turnkey.md#9` (expiring session keys)
- **Rationale:** The difference between bounded and unbounded blast radius is measured in transactions-per-revocation-lag.
- **Acceptance signal:** Revoke a sub-agent budget; within the next signing attempt, the sub-agent receives a typed denial; no successful signature produced between revoke and deny.

#### REQ-sa-delegation-as-policy-not-key: Delegation is never key-sharing

- **Statement:** No delegation flow exposed by the wallet requires the delegating actor to copy secret material to the delegate.
- **Classification:** MUST
- **Actors:** A2, A4
- **Sources:** T1, T4, `02-threat-model.md#6` (architectural implication 4), `research/external/_tier-2/crypto/tether-wdk.md#10`
- **Rationale:** Stated as an architectural implication of the threat model; any tool that violates this is out of scope.
- **Acceptance signal:** Every documented delegation path produces either a scoped on-chain policy, a scoped Soroban auth entry, or a scoped subaccount — never a shared key.

#### REQ-sa-session-keys: Expiring session keys

- **Statement:** The wallet supports creating expiring session signers scoped to a time window, using the OZ `ContextRule`'s `valid_until` ledger bound as the on-chain expiry mechanism.
- **Classification:** SHOULD
- **Actors:** A2, A4
- **Sources:** `research/external/crypto/turnkey.md#9`, `04-smart-accounts.md#2` (`valid_until` on `ContextRule`), T1, T4
- **Acceptance signal:** Create a session-scoped `ContextRule` with `valid_until` set to a ledger ~15 minutes in the future; an auth-entry submission after the `valid_until` ledger closes is rejected by `__check_auth` at the ledger layer (on-chain enforcement, not client-side).

#### REQ-sa-guard-hooks: Global guard / invariant hook (gap vs OZ)

- **Statement:** Pre-sign and post-execution global-invariant hooks (analogous to Safe's `Guard` trait) are not provided by OZ `stellar-accounts` v0.7.1. OZ policies are rule-scoped: each policy's `enforce()` runs only for the matched context rule. Expressing a global "never above M per period regardless of rule" invariant requires either (a) attaching a spending-limit policy to a `Default`-context rule that matches any operation, or (b) implementing a custom per-project guard-equivalent extension. This requirement records the gap and the recommended OZ-shaped workaround; it is not a claim that OZ ships a guard trait.
- **Classification:** SHOULD
- **Actors:** A2, A4
- **Sources:** `research/external/crypto/safe.md#9` (Safe's Guard trait, named here for contrast), `04-smart-accounts.md#7` (gap list), T2, T4
- **Acceptance signal:** A `Default`-context rule carrying a spending-limit policy rejects a transaction across every auth context in a test harness when the cap is exceeded; documented as the OZ-shaped substitute for Safe's Guard semantics.

### 1.5 SEP-based flows

#### REQ-sep-sep10-client: SEP-10 client authentication

- **Statement:** The wallet acts as a SEP-10 client: fetch challenge, sign, exchange for a token, and expose the token to the calling tool as a scoped capability.
- **Classification:** MUST
- **Actors:** A3
- **Sources:** A3 ("SEP-10 client including ephemeral keys"), `research/external/stellar-ecosystem/freighter-walletkit.md#9`
- **Rationale:** SEP-10 is the on-ramp for every Stellar-native service the agent transacts with.
- **Acceptance signal:** Authenticate against a reference SEP-10 server using a wallet identity; token returned, scoped, and revocable.

#### REQ-sep-sep10-ephemeral: Ephemeral auth-only keypairs

- **Statement:** The wallet supports using an ephemeral keypair for SEP-10 auth-only operations without funding or exposing it as a signing identity.
- **Classification:** SHOULD
- **Actors:** A3
- **Sources:** A3 ("ephemeral keys for auth-only operations"); G1 drained here (see §0.3.1)
- **Rationale:** The signing key should not be linkable to every provider the agent auths to.
- **Acceptance signal:** SEP-10 auth flow succeeds using a keypair that does not appear as a balance-holding identity and is not persisted past the session.

#### REQ-sep-sep43-walletkit: SEP-43 interface shape for Wallet Kit integration

- **Statement:** The wallet exposes a SEP-43-compatible interface so it can act as a Stellar Wallet Kit module.
- **Classification:** SHOULD
- **Actors:** A4, A7
- **Sources:** `research/external/stellar-ecosystem/freighter-walletkit.md#9`, `research/brain-dump/requirements.md:22`
- **Rationale:** Adoption surface: every dapp using Wallet Kit gets the wallet for free.
- **Acceptance signal:** Wallet registers as a Wallet Kit module in a reference dapp; user can connect and sign from the dapp's side.

#### REQ-sep-stellar-toml: `stellar.toml` resolution for counterparties

- **Statement:** The wallet resolves counterparty home domains and SEP-10 identity via `stellar.toml` and surfaces the result on approval prompts.
- **Classification:** SHOULD
- **Actors:** A3, A4
- **Sources:** T5, `02-threat-model.md#T5` defences
- **Rationale:** Identity-anchored allowlists are the T5 defence; address-only allowlists do not catch lookalike issuers.
- **Acceptance signal:** Sending to a federation-resolved address displays the resolved home domain and issuer identity on the approval prompt.

#### REQ-sep-out-of-scope-issuance: SEP-24 / SEP-6 deposit/withdraw flows out for v1

- **Statement:** The wallet does not implement SEP-24 or SEP-6 deposit-and-withdrawal flows in v1.
- **Classification:** OUT
- **Actors:** A4
- **Sources:** team-judgment, `analysis/00-context.md#3` (SEP coverage deferred)
- **Rationale:** Off-chain anchors are a separate concern from the self-custodial agent-wallet core; can be shipped as a skill later.
- **Acceptance signal:** Documented in §4.1; no SEP-24 flow code in v1.

### 1.6 Payments to services (x402, MPP, SEP-10 client flows)

#### REQ-svc-x402-consume: Consume x402-paywalled resources

- **Statement:** The wallet can complete an x402 payment handshake to unlock a paywalled HTTP resource on behalf of the agent, scoped by policy.
- **Classification:** MUST
- **Actors:** A3
- **Sources:** `research/brain-dump/requirements.md:5`, `analysis/00-context.md#5.4` (track 1), A3 ("native x402 flow")
- **Rationale:** x402 is SDF's stated track and one of two HTTP-402-based agent-pays-a-service surfaces in this analysis; the other is MPP (covered in `03-requirements-addendum.md` §5.10 and `research/stellar-capabilities/10-mpp.md`).
- **Acceptance signal:** A reference paid API responds with 402; wallet reads payment requirements, produces the Soroban auth entry, pays, and receives the resource in the second request.

#### REQ-svc-x402-nonce-binding: x402 confirmation is wallet-issued nonce, not agent boolean

- **Statement:** The two-step simulate-then-commit pattern for x402 payments ties commit to a wallet-issued nonce returned by simulate, not to an agent-provided boolean.
- **Classification:** MUST
- **Actors:** A3
- **Sources:** T2, T3, `research/external/_tier-2/crypto/phantom-mcp-server.md#10` (anti-pattern: agent-set `confirmed`), `research/external/_summary.md#cross-cutting finding 5`
- **Rationale:** This is the named differentiator from all other 2026 MCP wallets.
- **Acceptance signal:** Commit tool rejects a forged nonce; the nonce in a successful commit matches exactly the one returned by the prior simulate for the same payload hash.

#### REQ-svc-x402-receipt: Correlatable x402 receipts

- **Statement:** Each completed x402 payment returns a receipt correlating the on-chain transaction, the HTTP request, and the payment requirements.
- **Classification:** SHOULD
- **Actors:** A1, A3
- **Sources:** A3 ("receipts correlated with off-chain invoices")
- **Acceptance signal:** Receipt includes tx hash, request URL, resource identifier, amount, counterparty, and the payment-requirements digest.

#### REQ-svc-x402-out-provide: Providing paid APIs via x402 is out of v1

- **Statement:** The wallet does not ship x402 *server* primitives (advertising paid endpoints, verifying incoming payments) in v1.
- **Classification:** OUT
- **Actors:** A1
- **Sources:** `research/brain-dump/requirements.md:5` ("possibly provide"), team-judgment
- **Rationale:** Consume is load-bearing for A3; provide is a distinct workload (HTTP server, resource catalogue, refund logic) that can ship as a skill.
- **Acceptance signal:** Documented in §4.1.

### 1.7 DEX / AMM / liquidity

#### REQ-dex-manage-offer: Manage sell / buy offers

- **Statement:** The wallet can create, update, and delete Stellar DEX offers.
- **Classification:** SHOULD
- **Actors:** A1, A7
- **Sources:** `research/brain-dump/requirements.md:7`, team-judgment (core trading primitive)
- **Acceptance signal:** Create a buy offer, modify it, and delete it; offer IDs returned at each step.

#### REQ-dex-orderbook-read: Read orderbook state

- **Statement:** The wallet can return the current orderbook for a given asset pair.
- **Classification:** SHOULD
- **Actors:** A1, A6, A7
- **Sources:** `research/brain-dump/requirements.md:18`
- **Acceptance signal:** Orderbook query returns bids and asks for a testnet pair, with prices and amounts in typed fields.

#### REQ-dex-amm: AMM liquidity pool interactions

- **Statement:** The wallet can deposit to and withdraw from Stellar liquidity pools and read pool state.
- **Classification:** SHOULD
- **Actors:** A1
- **Sources:** `research/brain-dump/requirements.md:7` (trading includes LP), team-judgment
- **Acceptance signal:** Deposit to a testnet pool, read the pool share, withdraw; balances match.

#### REQ-dex-swap: Swap via path-payment or AMM

- **Statement:** The wallet exposes a single swap tool that routes through path-payment or AMM based on best outcome.
- **Classification:** SHOULD
- **Actors:** A1, A3
- **Sources:** `research/brain-dump/requirements.md:7`, team-judgment
- **Acceptance signal:** Swap tool returns a quote, executes it, and the receipt identifies whether the path-payment or AMM route was used.

### 1.8 Read-only / observation flows

#### REQ-obs-no-key-required: Read operations do not require a key

- **Statement:** All read-only tools (balance, orderbook, orderbook depth, account state, ledger lookup, contract simulation) succeed without any signing identity configured.
- **Classification:** MUST
- **Actors:** A6
- **Sources:** A6 ("clean separation of read operations")
- **Rationale:** A6's entire actor profile depends on not being forced to hold a key.
- **Acceptance signal:** Fresh install with no identity can query account balances, orderbook, and run a Soroban simulation.

#### REQ-obs-account-state: Fetch account state for any G or C address

- **Statement:** The wallet exposes an account-state tool for both G-accounts and C-accounts (contract instance storage read).
- **Classification:** SHOULD
- **Actors:** A6, A7
- **Sources:** team-judgment, A6
- **Acceptance signal:** Query returns signers and thresholds for a G-account; storage keys and values for a C-account.

#### REQ-obs-events-stream: Subscribe to Soroban contract events

- **Statement:** The wallet can subscribe to Soroban contract events for a named contract and emit them on stdout as newline-delimited JSON.
- **Classification:** SHOULD
- **Actors:** A6
- **Sources:** `research/external/non-crypto/stripe-cli.md#9` (`stripe listen --forward-to`), C6
- **Acceptance signal:** Running the subscribe tool against a test contract emits each event as a JSON line within 2 seconds of the emitting transaction.

## 2. Non-functional

### 2.1 Security and threat defence

#### REQ-sec-local-policy-engine: Local policy engine is mandatory

- **Statement:** A policy engine runs inside the wallet process and evaluates every proposed signature before the key is touched.
- **Classification:** MUST
- **Actors:** A1, A2, A4
- **Sources:** T2, T4, `02-threat-model.md#6` (architectural implication 1), `research/external/crypto/moonpay-agents-ows.md#9`, `research/external/crypto/safe.md#9`
- **Rationale:** The threat model names this an architectural implication; every candidate that puts policy vendor-side or in the agent prompt is disqualified.
- **Acceptance signal:** A policy-denied transaction never reaches the signer; the deny is logged with the draft it refused.

#### REQ-sec-policy-per-tx-cap: Per-transaction amount caps

- **Statement:** Policy rules support per-transaction amount caps in a USD-denominated criterion and in native-asset units.
- **Classification:** MUST
- **Actors:** A1, A4
- **Sources:** T2, T6, `research/brain-dump/requirements.md:6`, `research/external/crypto/coinbase-agentic.md#9` (`changeCents`), `research/external/crypto/trust-wallet-agent-kit.md#9` (`--max-usd`)
- **Rationale:** USD-denominated caps avoid asset-unit confusion (T3 mitigation); native caps cover assets without a reliable USD quote.
- **Acceptance signal:** A 1 USD cap rejects a 2 USD payment; a native-XLM cap rejects a payment above the cap regardless of USD quote availability.

#### REQ-sec-policy-per-period-cap: Per-period spending caps

- **Statement:** Policy rules support per-period (per-hour, per-day) cumulative spending caps.
- **Classification:** MUST
- **Actors:** A1, A2, A3
- **Sources:** T2, T6, A1 ("enforceable spending limits ... per-tx, per-period, per-counterparty")
- **Acceptance signal:** A daily-cap policy rejects the N+1 transaction on the day the cap is hit; counter resets on the declared window boundary.

#### REQ-sec-policy-counterparty-allowlist: Counterparty allowlists anchored on identity, not address

- **Statement:** Policy rules can allowlist counterparties by SEP-10 identity, stellar.toml home domain, or known issuer, as well as by raw address.
- **Classification:** MUST
- **Actors:** A3, A4
- **Sources:** T5, A3 ("identity-anchored policy"), `research/external/_tier-2/crypto/fireblocks.md#9` (typed object kinds)
- **Rationale:** Raw-address allowlists cannot distinguish a lookalike issuer; identity-anchored allowlists can.
- **Acceptance signal:** A policy that allowlists a home domain accepts payments to any address resolving to that domain, and rejects an address whose home domain does not match.

#### REQ-sec-policy-rate-limit: Rate limit signatures per identity

- **Statement:** Policy rules support rate limits on signatures per identity per window.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** T6, A1
- **Rationale:** Runaway-loop defence distinct from value caps.
- **Acceptance signal:** A rate limit of 10/min denies the 11th signature within 60 seconds with a typed error.

#### REQ-sec-policy-minimum-reserve: Minimum-reserve guard

- **Statement:** The wallet refuses to submit a transaction that would leave the account below its minimum reserve.
- **Classification:** SHOULD
- **Actors:** A1, A5
- **Sources:** T6, `02-threat-model.md#T6` defences
- **Rationale:** A classic footgun: draining the base reserve renders the account unusable. Per-capability arithmetic (base reserves, Soroban footprints, LP / DEX state) is absorbed into this REQ via the addendum §1 merge log (stream-2 sources from `01-accounts.md`, `02-classic-ops.md`, `07-dex-amm.md`); CAP-0073 final reserve model tracked in G3 (narrowed).
- **Acceptance signal:** A payment that would drop the source below base reserve is rejected pre-signing with a dedicated error code.

#### REQ-sec-policy-signed: Policy file is signed and verified

- **Statement:** The wallet refuses to load a policy file that is not signed by the identity that installed it.
- **Classification:** MUST
- **Actors:** A1, A2, A7
- **Sources:** T2, T8, `research/external/_tier-2/crypto/privy.md#9` (`owner_id` with mutation-signed-by-owner)
- **Rationale:** An unsigned policy file is equivalent to no policy for any attacker that can write to the config directory.
- **Acceptance signal:** Tampering with an installed policy file breaks the signature check; the wallet refuses to sign until policy is re-signed.

#### REQ-sec-dangerous-tool-gate: Dangerous tools require explicit acknowledgement

- **Statement:** Tools classified as destructive (fund drain, trustline removal on non-zero balance, account merge) require an explicit acknowledgement field and fail closed if it is absent.
- **Classification:** SHOULD
- **Actors:** A1
- **Sources:** T2, `research/external/crypto/kraken-cli.md#9` (dangerous-tool gate with `acknowledged=true`)
- **Acceptance signal:** Destructive tool invoked without the acknowledgement field returns a typed error and does not sign.

#### REQ-sec-typed-amounts: Typed amount schema with explicit unit

- **Statement:** Every amount field in every tool carries an explicit unit (`base` or `ui`) and an explicit `decimals` value.
- **Classification:** MUST
- **Actors:** A1, A2, A3, A7
- **Sources:** T3, `research/external/_tier-2/crypto/phantom-mcp-server.md#9` (dual-unit schema), `research/external/_tier-2/crypto/tether-wdk.md#9` (typed amount parsing)
- **Rationale:** Stroop-vs-XLM errors are the most common class of amount mistakes; making the unit ambiguity structurally impossible closes the class.
- **Acceptance signal:** Tool schemas reject amounts with missing unit; integration test that sends `"1"` with `unit=base` and `"1"` with `unit=ui` produces two different transactions.

#### REQ-sec-typed-amount-errors: Typed amount-parse error codes

- **Statement:** Amount parsing returns typed error codes (`EMPTY_STRING`, `INVALID_FORMAT`, `EXCESSIVE_PRECISION`) rather than a generic parse error.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** T3, `research/external/_tier-2/crypto/tether-wdk.md#9`
- **Acceptance signal:** Each error class is produced by a dedicated test and routed by an agent integration test to a different recovery path.

#### REQ-sec-network-required: Network is an explicit required parameter everywhere

- **Statement:** Every tool requires an explicit `network` parameter (or inherits from a structurally declared profile); no ambient default exists at the process level.
- **Classification:** MUST
- **Actors:** A5, A7
- **Sources:** T10, `02-threat-model.md#6` (architectural implication 5), `research/external/stellar-ecosystem/stellar-mcp-server.md#10`
- **Rationale:** Network confusion is a design-smell threat that must be structurally impossible.
- **Acceptance signal:** No tool accepts an operation without network either in the call or in the profile; changing network mid-session is not silent.

#### REQ-sec-network-marker: Network is shown on every response

- **Statement:** Every tool response includes the network the operation ran against.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** T10, `research/external/crypto/kraken-cli.md#9` (paper-mode marker), `research/external/stellar-ecosystem/stellar-cli.md#9`
- **Acceptance signal:** JSON output carries a `network` field on every response; human-readable output prefixes every line with a network marker.

#### REQ-sec-network-isolation: Separate keystores per network

- **Statement:** Identity profiles scoped to testnet and mainnet use distinct keystore namespaces.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** T10, `02-threat-model.md#T10` defences
- **Rationale:** Eliminates the "oops this was the mainnet identity" class by construction.
- **Acceptance signal:** Creating a testnet identity with the same name as a mainnet identity does not overwrite it; addresses differ.

#### REQ-sec-rpc-multi-endpoint: Configurable multiple RPC endpoints

- **Statement:** The wallet accepts multiple RPC endpoints per network and exposes which endpoint produced each piece of state.
- **Classification:** SHOULD
- **Actors:** A1, A5
- **Sources:** T7, `02-threat-model.md#T7` defences
- **Rationale:** Partial T7 defence even before cross-check (C10) is promoted.
- **Acceptance signal:** Response carries an `endpoint` field; switching endpoint emits a warning.

#### REQ-sec-wallet-owned-approval: Approval UX is wallet-controlled

- **Statement:** Approval prompts for high-value or policy-escalated actions are rendered by the wallet, not by the agent.
- **Classification:** MUST
- **Actors:** A4, A7
- **Sources:** T9, `02-threat-model.md#6` (architectural implication 3), `research/external/crypto/trust-wallet-agent-kit.md#9` (WalletConnect approval transport)
- **Rationale:** If the agent renders the prompt, it picks the dangerous field out of the display; T9 is unaddressed.
- **Acceptance signal:** Approval bytes reach the user through a channel the agent cannot forge; no approval can be granted by the agent printing a string.

#### REQ-sec-approval-highlight-changes: Approval UX highlights novel fields

- **Statement:** The approval prompt highlights fields that differ from recent history for the same counterparty class.
- **Classification:** SHOULD
- **Actors:** A4, A7
- **Sources:** T9, `02-threat-model.md#T9` defences
- **Acceptance signal:** First-time destination for a given identity is flagged visually distinctly from a repeat destination.

#### REQ-sec-approval-nonce: Approvals carry wallet-issued nonces

- **Statement:** Each approval response binds to a wallet-issued nonce and cannot be replayed.
- **Classification:** SHOULD
- **Actors:** A4
- **Sources:** T9, `research/external/_tier-2/crypto/phantom-mcp-server.md#10` + `#9` (nonce-bound confirmation)
- **Acceptance signal:** Replaying a previous approval envelope is rejected; changing the payload changes the nonce.

#### REQ-sec-skill-isolation: Skills run isolated from keystore

- **Statement:** Installed skills / plugins run in a sandbox that cannot read keys, cannot modify in-flight transactions, and cannot silently mutate policy.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** T8, `research/external/_tier-2/crypto/metamask-snaps.md#9` (SES compartments)
- **Rationale:** The extensibility story must not be the threat vector that dominates.
- **Acceptance signal:** A reference malicious skill that attempts to read the keystore or re-sign a transaction is refused at sandbox boundary.

#### REQ-sec-skill-signed: Skills are signed and pinned

- **Statement:** Skills install by `(package, version, shasum)` tuple and by signature from a declared publisher.
- **Classification:** SHOULD
- **Actors:** A1, A7
- **Sources:** T8, `research/external/_tier-2/crypto/metamask-snaps.md#9`, `research/external/crypto/safe.md#9` (Rhinestone Registry)
- **Acceptance signal:** Install refuses a tampered tarball; install refuses a publisher key not on the configured trust set.

#### REQ-sec-skill-capability-manifest: Declarative capability manifest per skill

- **Statement:** Each skill declares the wallet capabilities it requires in a manifest, and the wallet refuses to grant undeclared capabilities at runtime.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** T8, `research/external/_tier-2/crypto/metamask-snaps.md#9` (`initialPermissions`)
- **Acceptance signal:** A skill invoked to sign without declaring signing capability is refused.

#### REQ-sec-skill-first-invoke-gate: First-invoke gate on signing capabilities

- **Statement:** A skill requesting a signing capability triggers a user-approval prompt the first time it is invoked, not only at install.
- **Classification:** SHOULD
- **Actors:** A4, A7
- **Sources:** T2, T8, `research/external/_tier-2/crypto/metamask-snaps.md#10` (install-time-only consent is insufficient)
- **Acceptance signal:** Install does not itself grant signing; the first sign invocation requires user confirmation.

#### REQ-sec-audit-log-signed: Audit log is hash-chained

- **Statement:** The audit log is append-only with each entry cryptographically chained to its predecessor.
- **Classification:** SHOULD
- **Actors:** A1, A2, A7
- **Sources:** T8; G6 drained here (see §0.3.1)
- **Rationale:** Tamper-evidence is what makes the audit log useful for post-incident analysis. Hash-chained format chosen — drains G6.
- **Acceptance signal:** Modifying any past audit entry breaks chain verification.

#### REQ-sec-short-unlock: Short in-memory unlock windows

- **Statement:** The wallet holds decrypted key material in memory only for the duration of a named unlock window, not indefinitely.
- **Classification:** SHOULD
- **Actors:** A1, A5, A7
- **Sources:** T1, `02-threat-model.md#T1` defences, `research/external/_tier-2/crypto/tether-wdk.md#9` (`.dispose()` lifecycle)
- **Rationale:** Limits post-compromise exposure window.
- **Acceptance signal:** After the declared unlock TTL, an attempt to sign without re-unlock returns a typed `auth_required` error.

#### REQ-sec-policy-mutation-governance: Policy mutations require owner signature

- **Statement:** Every mutation to policy files or allowlists requires a signature from the declared owner identity.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** T2, T8, `research/external/_tier-2/crypto/privy.md#9`, `research/external/_tier-2/crypto/fireblocks.md#9` (Admin-Quorum)
- **Acceptance signal:** Agent-initiated policy edits are rejected; only owner-signed edits are accepted.

### 2.2 Usability (agent caller, human operator, companion UI)

#### REQ-ux-json-default: JSON is the default machine output

- **Statement:** Every command supports `--json` and emits schema-stable machine-readable output.
- **Classification:** MUST
- **Actors:** A1, A2, A5, A7
- **Sources:** N5, `research/brain-dump/requirements.md:12`, `research/external/crypto/kraken-cli.md#9`, `research/external/non-crypto/gh-cli.md#10` (stellar-cli's partial `--json` is the anti-pattern)
- **Rationale:** Named as a non-negotiable; agents cannot screen-scrape reliably.
- **Acceptance signal:** 100% of tools emit valid JSON under `--json`; a schema test covers every tool response.

#### REQ-ux-json-envelope: Uniform JSON envelope `{ok, data|error, request_id}`

- **Statement:** All machine-readable responses use a uniform envelope with `ok`, `data` or `error`, and `request_id` fields.
- **Classification:** MUST
- **Actors:** A1, A2
- **Sources:** `research/external/crypto/kraken-cli.md#9`, `research/external/non-crypto/aws-cli.md#10` (per-service drift anti-pattern)
- **Rationale:** Avoids per-tool response drift that makes agents brittle.
- **Acceptance signal:** Schema test: every response validates against the envelope schema.

#### REQ-ux-table-render: Human-readable table output mode

- **Statement:** Every command supports `--output table` with clean, pipe-safe column rendering.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** `research/brain-dump/requirements.md:13`, A7
- **Rationale:** The human operator is never worse off than the agent (A7); table output derives from JSON, not the other way round.
- **Acceptance signal:** `--output table` on every command produces a stable column order independent of terminal width.

#### REQ-ux-query-projection: `--query` / `--jq` projection

- **Statement:** Machine output supports a projection expression (`--query` or `--jq`) to subset fields without custom parsing.
- **Classification:** SHOULD
- **Actors:** A1, A5
- **Sources:** `research/external/non-crypto/gh-cli.md#9`, `research/external/non-crypto/aws-cli.md#9` (JMESPath)
- **Acceptance signal:** A documented projection returns only requested fields; exit code distinguishes empty projection from error.

#### REQ-ux-exit-codes: Standardised exit codes

- **Statement:** The wallet uses distinct exit codes for success (0), generic error (1), usage error (2), auth-required (4), policy-denied (5), and network error (6).
- **Classification:** SHOULD
- **Actors:** A1, A5
- **Sources:** `research/brain-dump/requirements.md:16`, `research/external/non-crypto/gh-cli.md#9`, `research/external/crypto/kraken-cli.md#10` (single-exit-code anti-pattern)
- **Acceptance signal:** Each class is triggered by an integration test and produces the documented exit code.

#### REQ-ux-error-taxonomy: Error envelope with typed category

- **Statement:** Error responses carry a `category` enum the agent can route on, not free-form messages.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** `research/brain-dump/requirements.md:16`, `research/external/crypto/kraken-cli.md#9` (nine-category envelope)
- **Rationale:** Agents and scripts should not regex error strings.
- **Acceptance signal:** Every error path produces a documented category; categories are enumerated in the schema.

#### REQ-ux-idempotency-key: Idempotency key on every mutation

- **Statement:** Every mutating tool accepts an idempotency key and returns the same response on retry within a documented window.
- **Classification:** SHOULD
- **Actors:** A1, A5
- **Sources:** `research/external/non-crypto/stripe-cli.md#9`, A5 ("idempotent operations")
- **Rationale:** Sequence-number collisions and network retries make a naive retry dangerous; idempotency keys let the wallet safely deduplicate.
- **Acceptance signal:** Retrying a payment with the same idempotency key returns the first response, not a second submission.

#### REQ-ux-mcp-stdio: Built-in MCP server over stdio

- **Statement:** The wallet ships an MCP server that speaks over stdio and is launched by the same binary.
- **Classification:** MUST
- **Actors:** A1, A2, A3, A7
- **Sources:** `research/brain-dump/requirements.md:17`, `research/external/crypto/kraken-cli.md#9`, `research/external/_tier-2/crypto/phantom-mcp-server.md#9`
- **Rationale:** Stated directly in the brain-dump; the MCP-stdio shape is the convergent pattern across every 2026 agent wallet.
- **Acceptance signal:** `stellar-ai mcp` starts a working MCP server on stdio; a reference MCP client enumerates tools and calls `balance`.

#### REQ-ux-mcp-tool-annotations: MCP tools carry read/destructive/idempotent hints

- **Statement:** Every MCP tool has `readOnlyHint`, `destructiveHint`, `idempotentHint`, and `openWorldHint` annotations.
- **Classification:** SHOULD
- **Actors:** A1, A2, A7
- **Sources:** `research/external/_tier-2/crypto/tether-wdk.md#9`, `research/external/_tier-2/crypto/phantom-mcp-server.md#9`
- **Rationale:** Lets an agent runtime pre-filter tools that are safe to call without approval.
- **Acceptance signal:** Every registered tool carries the four annotations; the MCP schema reports them.

#### REQ-ux-mcp-tool-names-namespaced: Chain-namespaced tool names

- **Statement:** MCP tool names are namespaced with `stellar_` and every tool schema includes a CAIP-2 identifier.
- **Classification:** SHOULD
- **Actors:** A1, A7
- **Sources:** T10, `research/external/_tier-2/crypto/phantom-mcp-server.md#9`
- **Rationale:** Structural defence against cross-chain tool confusion when multiple wallet MCP servers run in the same agent.
- **Acceptance signal:** All tool names start with `stellar_`; tool schemas include `stellar:pubnet` or `stellar:testnet` as appropriate.

#### REQ-ux-mcp-read-write-groups: Typed read/write tool groupings

- **Statement:** MCP exposes a read-only tool subset that can be selected at registration time, independent of the write subset.
- **Classification:** SHOULD
- **Actors:** A6
- **Sources:** `research/external/_tier-2/crypto/tether-wdk.md#9` (`WALLET_READ_TOOLS` / `WALLET_WRITE_TOOLS`)
- **Acceptance signal:** Starting MCP with `--tools read` exposes only read-only tools; a sign attempt returns tool-not-found.

#### REQ-ux-setup-wizard: Interactive setup wizard

- **Statement:** The wallet provides an interactive setup command that walks a human through identity creation, keyring storage, and default network selection.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** `research/brain-dump/requirements.md:24`, A7 ("discoverability matters")
- **Acceptance signal:** Running the wizard on a fresh install leaves a working identity and config file without any other commands.

#### REQ-ux-dry-run: Dry-run on every mutating command

- **Statement:** Every mutating command supports `--dry-run` that returns the resolved transaction and policy decisions without signing.
- **Classification:** SHOULD
- **Actors:** A5, A7
- **Sources:** T3, A5 ("dry-run mode"), `02-threat-model.md#T3` defences
- **Acceptance signal:** `--dry-run` on a payment returns the exact XDR that would be signed, plus the policy decision tree, and never submits.

#### REQ-ux-companion-ui: Companion UI for approval and funding

- **Statement:** A companion UI accepts asynchronous approval requests and initiates funding flows via Stellar Wallet Kit or WalletConnect.
- **Classification:** SHOULD
- **Actors:** A4, A7
- **Sources:** `research/brain-dump/requirements.md:11`, `research/brain-dump/requirements.md:22`, A4 ("asynchronous approval channel"), T9
- **Rationale:** A4's actor profile collapses without an async approval surface; T9 requires wallet-owned rendering.
- **Acceptance signal:** Companion UI receives an approval request, displays it to the user, returns a signed approval bound to a nonce; WalletConnect and Wallet Kit funding round-trips succeed on testnet.

### 2.3 Observability and audit

#### REQ-audit-stderr-log: Structured stderr audit log

- **Statement:** The wallet emits a structured audit log to stderr carrying agent identifier, instance, pid, transport, and argument *keys only*.
- **Classification:** SHOULD
- **Actors:** A1, A2, A7
- **Sources:** `research/external/crypto/kraken-cli.md#9`, A1 ("auditable log of what was signed and why")
- **Rationale:** Keys-only protects secret values from accidental log exfiltration.
- **Acceptance signal:** Log entries contain the documented fields; no secret value (seed, password) appears in any log line in a fuzzing test.

#### REQ-audit-policy-denials: Every policy denial is logged with the draft it refused

- **Statement:** When policy denies a proposed signature, the audit log records the policy rule that denied it and the resolved draft transaction.
- **Classification:** SHOULD
- **Actors:** A1, A2, A7
- **Sources:** T2, `02-threat-model.md#T2` defences
- **Acceptance signal:** A deny produces a log line with the rule ID, the caller, and the XDR it refused.

#### REQ-audit-approval-events: Approval events logged with nonce

- **Statement:** Every approval (granted or denied) produces an audit log event including the wallet-issued nonce.
- **Classification:** SHOULD
- **Actors:** A4, A7
- **Sources:** T9, REQ-sec-approval-nonce
- **Acceptance signal:** Approval events are queryable by nonce; nonce matches the one bound to the signature.

#### REQ-audit-receipts-human-retrievable: Human-retrievable receipts

- **Statement:** The wallet stores a per-transaction receipt the user can retrieve by transaction hash or by date range.
- **Classification:** SHOULD
- **Actors:** A1, A7
- **Sources:** team-judgment (operational usefulness), A1
- **Acceptance signal:** Query by hash returns the receipt; query by date range returns all receipts in the window.

### 2.4 Performance and scale

#### REQ-perf-latency-signing: Signing latency under 500 ms

- **Statement:** Typical classic-payment signing (policy evaluated + signed + submit prepared) completes in under 500 ms on reference hardware.
- **Classification:** SHOULD
- **Actors:** A1, A3
- **Sources:** team-judgment, A3 ("frequent, very small")
- **Rationale:** x402 and service-consumer flows are latency-sensitive; a slow wallet becomes the critical path.
- **Acceptance signal:** Benchmark emits signing latency; p50 < 500 ms on the reference laptop.

#### REQ-perf-sequence-pool-throughput: Documented sequence-pool throughput

- **Statement:** The in-process sequence pool documents sustainable transactions-per-second per identity.
- **Classification:** SHOULD
- **Actors:** A1
- **Sources:** `research/external/_tier-2/stellar-ecosystem/launchtube.md#9`, A1
- **Acceptance signal:** Published benchmark with reproducible script; number documented in spec.

#### REQ-perf-resource-caps: Per-skill CPU and memory caps

- **Statement:** Skills run under documented CPU time and memory caps that fail closed when exceeded.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** T6, T8, `research/external/_tier-2/crypto/metamask-snaps.md#10` (`endowment:long-running` anti-pattern)
- **Acceptance signal:** A skill that exceeds its cap is terminated; the wallet remains responsive.

### 2.5 Extensibility (skills, plugins)

#### REQ-ext-skills-pattern: Skill / plugin extensibility

- **Statement:** The wallet supports user-installable skills that add new tools without modifying the core binary.
- **Classification:** SHOULD
- **Actors:** A1, A2
- **Sources:** `research/brain-dump/requirements.md:8`, `research/external/stellar-ecosystem/stellar-cli.md#9` (external-binary plugin pattern), `research/external/_tier-2/crypto/metamask-snaps.md#9`
- **Acceptance signal:** Installing a reference skill registers new MCP tools and new CLI subcommands; uninstall removes both cleanly.

#### REQ-ext-skill-audit-gate: Audit requirement gating key-touching skills

- **Statement:** Skills that request signing or key-derivation capability require a third-party audit attestation before they can be installed from the default registry.
- **Classification:** SHOULD
- **Actors:** A1, A7
- **Sources:** T8, `research/external/_tier-2/crypto/metamask-snaps.md#9`, `research/external/crypto/safe.md#9`
- **Acceptance signal:** A skill requesting signing without attestation is refused by the default registry; users can override with a named flag.

#### REQ-ext-library-crate: Embedding library crate

- **Statement:** Core wallet functionality is available as a library (Rust crate or equivalent) for embedding in other tools.
- **Classification:** SHOULD
- **Actors:** A5, A7
- **Sources:** `research/external/stellar-ecosystem/stellar-cli.md#9` (public `soroban_cli` crate)
- **Acceptance signal:** A downstream binary imports the library and calls `submit` without depending on a separate process.

## 3. Delivery and operations

### 3.1 Packaging and distribution

#### REQ-pkg-prebuilt-binaries: Pre-built binaries on GitHub releases

- **Statement:** Each release ships pre-built binaries for macOS (arm64 and x86_64) and Linux (x86_64 and arm64) on the GitHub releases page.
- **Classification:** MUST
- **Actors:** A5, A7
- **Sources:** `research/brain-dump/requirements.md:15`
- **Acceptance signal:** `gh release view <tag>` lists four target binaries; each installs and runs `--version` successfully.

#### REQ-pkg-reproducible-builds: Reproducible builds

- **Statement:** Release binaries are reproducible from source with a documented process.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** T8, `02-threat-model.md#T8` defences
- **Rationale:** Reproducible-build evidence lets independent parties verify releases; partial mitigation against a compromised CI pipeline.
- **Acceptance signal:** Running the documented build on a second environment produces a binary with the same hash.

#### REQ-pkg-signed-releases: Signed releases

- **Statement:** Release artifacts are signed and the public key is documented in the repository.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** T8
- **Acceptance signal:** Each release artifact has a detached signature verifiable against the documented key.

#### REQ-pkg-permissive-licence: Permissive open-source licence

- **Statement:** The wallet, reference skills, and library crate are released under a permissive licence compatible with commercial adoption.
- **Classification:** MUST
- **Actors:** A7
- **Sources:** N4, `research/external/crypto/safe.md#10` (LGPL anti-pattern)
- **Rationale:** Named as a non-negotiable; Safe's LGPL-3.0 is the avoid-reason cited in research.
- **Acceptance signal:** `LICENSE` file names Apache-2.0 or MIT; no dependency imposes a conflicting copyleft.

### 3.2 Platform support

#### REQ-plat-macos: macOS support (arm64 and x86_64)

- **Statement:** The wallet builds and passes tests on macOS arm64 and x86_64.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** `research/brain-dump/requirements.md:14`
- **Acceptance signal:** CI runs on both architectures and all tests pass.

#### REQ-plat-linux: Linux support (x86_64 and arm64)

- **Statement:** The wallet builds and passes tests on Linux x86_64 and arm64.
- **Classification:** MUST
- **Actors:** A1, A5
- **Sources:** `research/brain-dump/requirements.md:14`
- **Acceptance signal:** CI runs on both architectures and all tests pass.

#### REQ-plat-windows: Windows support

- **Statement:** Windows is not a v1 platform target; contributions welcome.
- **Classification:** OUT
- **Actors:** A7
- **Sources:** `research/brain-dump/requirements.md:14` (macos + linux only), team-judgment
- **Rationale:** A7 and A1 concentrate on Unix hosts; Windows-DPAPI keyring support is a known-cost extension post-v1.
- **Acceptance signal:** Documented in §4.1.

#### REQ-plat-testnet-mainnet-parity: Testnet and mainnet parity

- **Statement:** Every feature works on both testnet and mainnet without separate code paths.
- **Classification:** MUST
- **Actors:** A1, A7
- **Sources:** N6, `research/brain-dump/requirements.md:20`
- **Rationale:** Non-negotiable; `research/external/stellar-ecosystem/stellar-mcp-server.md#10` documents the anti-pattern this requirement rejects (hardcoded testnet URL).
- **Acceptance signal:** Integration tests run on both networks; a feature matrix shows no network-specific exclusions.

### 3.3 Configuration and secrets

#### REQ-cfg-two-layer: Non-secret config + keyring secrets

- **Statement:** Configuration is split between a non-secret TOML file and an OS-keyring-backed secret store, both keyed by the same profile name.
- **Classification:** MUST
- **Actors:** A5, A7
- **Sources:** `research/brain-dump/requirements.md:19`, `research/external/non-crypto/aws-cli.md#9`, `research/external/stellar-ecosystem/stellar-cli.md#9`
- **Rationale:** AWS CLI's two-layer split is the convergent pattern; Stripe and stellar-cli follow it.
- **Acceptance signal:** Config file contains no secret material; keyring query returns a secret under a profile-named entry.

#### REQ-cfg-xdg: XDG-compliant config location

- **Statement:** The wallet stores configuration under `$XDG_CONFIG_HOME/stellar-ai/` (or platform equivalent) with directory mode `0o700`.
- **Classification:** SHOULD
- **Actors:** A5, A7
- **Sources:** `research/external/non-crypto/gh-cli.md#9`, `research/external/stellar-ecosystem/stellar-cli.md#9`
- **Acceptance signal:** Fresh install creates the documented directory with the documented mode.

#### REQ-cfg-env-vars: Environment-variable overrides

- **Statement:** Every config option has a corresponding environment variable with a documented precedence chain.
- **Classification:** SHOULD
- **Actors:** A1, A5
- **Sources:** `research/brain-dump/requirements.md:19`, `research/external/non-crypto/aws-cli.md#9`, A5 ("environment-variable configuration")
- **Acceptance signal:** Precedence documented as flag > env > profile-file > default; a test per layer verifies the chain.

#### REQ-cfg-profile-selector: Named profile selector env var

- **Statement:** A `STELLAR_WALLET_PROFILE` (or equivalent) environment variable selects the active identity profile.
- **Classification:** SHOULD
- **Actors:** A5
- **Sources:** `research/external/non-crypto/aws-cli.md#9` (`AWS_PROFILE`)
- **Acceptance signal:** Setting the env var switches the default identity without a flag.

#### REQ-cfg-noninteractive-unlock: Non-interactive unlock path

- **Statement:** A documented mechanism unlocks the keystore without a terminal prompt for CI use.
- **Classification:** SHOULD
- **Actors:** A5
- **Sources:** A5 ("non-interactive unlock")
- **Acceptance signal:** A CI-only invocation using a documented env variable or passphrase file signs successfully without a TTY.

### 3.4 Telemetry posture

#### REQ-tel-no-default: Telemetry is off by default

- **Statement:** The wallet emits no telemetry to any third party unless the user explicitly opts in.
- **Classification:** MUST
- **Actors:** A7
- **Sources:** N3, `analysis/00-context.md#4`
- **Rationale:** Non-negotiable; default-off is the only posture compatible with self-custodial framing.
- **Acceptance signal:** A network-capture during normal operation shows no outbound traffic to project-operated endpoints.

#### REQ-tel-opt-in-non-identifying: Opt-in telemetry is non-identifying

- **Statement:** Any opt-in telemetry carries no user identifier, no address, and no transaction content; only aggregate counters.
- **Classification:** SHOULD
- **Actors:** A7
- **Sources:** N3
- **Acceptance signal:** Documented telemetry schema contains no field derivable to a user or address; a test verifies no such field is emitted.

## 4. Constraints (non-goals, explicit OUT)

### 4.1 Out-of-scope features

- **SEP-24 / SEP-6 deposit / withdrawal flows in v1.** Off-chain anchor integration is a separate surface from the self-custodial agent-wallet core; can ship as a skill. (REQ-sep-out-of-scope-issuance)
- **Providing x402-paywalled APIs.** Consume is core; provide is a distinct HTTP-server workload better left to skills. (REQ-svc-x402-out-provide)
- **Windows as a v1 platform target.** Unix-host actors dominate; Windows support is known-cost post-v1. (REQ-plat-windows)
- **Cloud-resident session keys / server-side policy engines / custodial agent accounts.** Disqualified by N1-N3; listed here so the question does not reopen.
- **Central server holding credentials, history, or policies.** Disqualified by N3.
- **Auto-updating binary without user consent.** Conflicts with signed-release verification expectations.
- **LLM inside the wallet.** The wallet does not embed a model; it is the subject of agent calls, not an agent itself.
- **Custodial recovery service operated by the project.** Disqualified by N2 / N3; if an optional recovery mode lands later, the server must be user-operated (see C3).
- **Raw-payload (`raw_sign`-style) signing with no Stellar semantic awareness.** Policy cannot reason about opaque bytes; excluded by design. (C9)
- **Free-form transaction-XDR submission as the only mutating surface.** Would widen T3 considerably; the wallet exposes typed construction paths as the default. (Narrow XDR-submit remains available as an escape hatch under an explicit flag; this does not make it the primary surface.)

### 4.2 Out-of-scope threat model items

Mirrors `02-threat-model.md#2.2`. Listed here so RFP readers see non-goals directly in the requirements document.

- **Host compromise with root privileges.** Minimised (short unlock windows, hardware-backed storage) but not claimed defeated.
- **Compromise of wallet signed-release artifacts.** Reproducible builds and signed releases mitigate; post-hoc detection of a compromised signing key is not claimed.
- **Colluding human operator.** Wallet defends against the agent, not the user who owns the key.
- **Sidechannel attacks on hardware wallets.** Delegated to the hardware vendor.
- **Chain-level attacks (51%, validator collusion).** Delegated to the Stellar network.
- **Denial of service at the RPC layer.** Mitigated by multi-endpoint configuration (REQ-sec-rpc-multi-endpoint); not guaranteed.
- **Deanonymisation via on-chain graph analysis.** On-chain activity is public by design.

---

## Appendix A: Seed mining from `research/brain-dump/requirements.md`

`research/brain-dump/requirements.md` contains 22 bullets (confirmed by file read; the scaffolding's row list over-counted; 23 and 24 are consolidated with 21 and 22 below). Each bullet is converted, split, or rejected below.

| # | Text (abbrev.) | Disposition |
|---|---|---|
| 01 | self-custody wallet controlled by AI agent (key isolation) | **rejected (partially)**: self-custody restates N1-N3 and lives in `00-context.md#4`; the "key isolation" half becomes REQ-acct-key-isolation |
| 02 | send payments on chain | → REQ-classic-payment |
| 03 | pay for external services via x402 (consume + possibly provide) | **split**: consume → REQ-svc-x402-consume, REQ-svc-x402-nonce-binding, REQ-svc-x402-receipt; provide → OUT (REQ-svc-x402-out-provide) |
| 04 | configurable spending limits (per-transaction) | **split**: → REQ-sec-policy-per-tx-cap + REQ-sec-policy-per-period-cap + REQ-sec-policy-counterparty-allowlist + REQ-sec-policy-rate-limit |
| 05 | trading | **split**: → REQ-classic-path-payment + REQ-dex-manage-offer + REQ-dex-amm + REQ-dex-swap |
| 06 | skills + skill extensibility | **split**: → REQ-ext-skills-pattern + REQ-sec-skill-isolation + REQ-sec-skill-signed + REQ-sec-skill-capability-manifest + REQ-sec-skill-first-invoke-gate + REQ-ext-skill-audit-gate |
| 07 | balances | → REQ-classic-balances |
| 08 | get wallet address | → REQ-acct-address-export |
| 09 | wallet companion UI | → REQ-ux-companion-ui |
| 10 | json output for commands `--json` | **split**: → REQ-ux-json-default + REQ-ux-json-envelope |
| 11 | human-friendly `--output table` rendering | → REQ-ux-table-render |
| 12 | macOS + Linux support | **split**: → REQ-plat-macos + REQ-plat-linux (+ REQ-plat-windows as OUT) |
| 13 | pre-built binaries on GitHub releases | → REQ-pkg-prebuilt-binaries |
| 14 | error envelopes, predictable exit codes | **split**: → REQ-ux-exit-codes + REQ-ux-error-taxonomy |
| 15 | built-in MCP server over stdio | **split**: → REQ-ux-mcp-stdio + REQ-ux-mcp-tool-annotations + REQ-ux-mcp-tool-names-namespaced + REQ-ux-mcp-read-write-groups |
| 16 | orderbook | → REQ-dex-orderbook-read |
| 17 | config file + environment variables | **split**: → REQ-cfg-two-layer + REQ-cfg-xdg + REQ-cfg-env-vars + REQ-cfg-profile-selector |
| 18 | testnet + mainnet support | → REQ-plat-testnet-mainnet-parity (restates N6; kept because it is the operational-level statement) |
| 19 | G-account + C-smart-account support | **split**: G-account is implicit across REQ-classic-*; C-account → REQ-sa-c-account-support + REQ-sa-policy-initiated-exec + REQ-sa-webauthn-signer + REQ-sa-guard-hooks |
| 20 | funding via companion UI (Stellar Wallet Kit, WalletConnect) | **merged into** REQ-ux-companion-ui + REQ-sep-sep43-walletkit |
| 21 | subaccounts (derived from mnemonic) | → REQ-acct-subaccounts (+ REQ-acct-subaccount-isolation for C-account-per-child, addendum §3; G4 drained) |
| 22 | setup wizard | → REQ-ux-setup-wizard |

**Disposition summary:** 22 source bullets; 18 converted (some with splits); 4 split into multiple requirements; 1 rejected-partially (bullet 01) with the justification that it restates non-negotiables.

## Appendix B: Open drafting rules

- Prefer `MUST` sparingly. A document that marks everything MUST is indistinguishable from no prioritisation.
- Every `MUST` requires a rationale of at least one sentence.
- Do not use the word "comprehensive" (project convention).
- Do not insert emojis (project convention).
- Current date stamp on edits: 2026-04-21.
- Stream-2 reconciliation is captured in the **addendum at `analysis/03-requirements-addendum.md`** (compact form — 36 merges, 2 splits, 106 adds, 2 promotions, 1 candidates-queue resolution; total 214 requirements). Downstream analysis docs cite requirements by the IDs in the addendum alongside the IDs in this file. Full-format expansion of new entries is deferred.
