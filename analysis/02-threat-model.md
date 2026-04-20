# Threat Model

**Status:** draft — 2026-04-20
**Prerequisite for:** `03-requirements.md`, `04-axes.md`, `06`, `07`
**Depends on:** `01-actor-model.md`

---

## 1. Defining frame

> **The agent is hostile by default.**

This frame is the most important choice in the document. It is not a claim that the user's agent is malicious; it is a claim about *what the wallet must assume*.

An LLM-driven process can be manipulated into issuing arbitrary tool calls by content it processes: an email it reads, a page it fetches, a document in its context, a response from another agent. This is prompt injection, and as of 2026 there is no general defence against it at the model level. Treating the agent as trusted makes every RAG document, every scraped page, and every upstream service a potential attacker against the wallet.

Consequence: the wallet's job is to **protect the user from their own agent**, not just from external attackers. Every downstream architecture decision flows from this frame.

This is a sharper constraint than a typical "zero trust" posture because the untrusted party is the *immediate caller*. The wallet cannot rely on the agent to sanitise inputs, bound actions, or distinguish user intent from injected instructions. The wallet itself must do these things before any signature is produced.

## 2. Operating assumptions

### 2.1 Threats in scope

- A compromised or misaligned agent process that constructs and submits malicious transactions through the wallet's interface.
- Prompt injection via any data channel the agent reads.
- Hallucinated tool calls: wrong arguments, wrong account, wrong network.
- Lateral attacks across sub-agents sharing a host, where one sub-agent attempts to exceed its delegated authority.
- Counterparty impersonation (lookalike domains, spoofed SEP-10 identities, memo-field phishing).
- Network-level tampering with RPC responses that could confuse a signer into approving a misrepresented transaction.
- Supply-chain compromise of the wallet's dependencies (within reason; see §2.2).
- Runaway loops that exhaust balance through many valid-looking small transactions.
- Smart-account-layer attacks that exploit the on-chain authorization model: rule-ID downgrade by a malicious transaction sponsor, signer-set-divergence rendering threshold policies unreachable, policy or verifier contract replacement at a deployment address, and cross-account compromise of a shared verifier contract.

### 2.2 Out-of-scope threats

Stating these explicitly keeps the analysis honest.

- **Host compromise with root privileges.** If the attacker owns the machine, they own any key the wallet can unlock. Exposure is minimised (short unlock windows, hardware-backed storage where available) but the wallet does not claim to defeat root.
- **Compromise of the wallet's signed release artifacts.** Reproducible-build and release-signing practice are followed; the wallet does not claim to detect a compromised signing key post hoc.
- **Colluding human operator.** If the legitimate user wants to drain the account, they can. The wallet defends against the agent, not the user.
- **Sidechannel attacks on hardware wallets.** Delegated to the hardware vendor.
- **Chain-level attacks** (51%, validator collusion). Delegated to the Stellar network.
- **Denial of service at the RPC layer.** Mitigated by multi-endpoint configuration; not guaranteed.
- **Deanonymisation via on-chain graph analysis.** On-chain activity is public by design.

## 3. Trust boundaries

```
+----------------------------------------------------------+
| User / Human Operator        [TRUSTED, root of trust]   |
+------------------------------+---------------------------+
                               |
                               v  configures, unlocks, approves high-value
+----------------------------------------------------------+
| Wallet core                  [TRUSTED, integrity-critical]|
|   - key store                                             |
|   - signer                                                |
|   - policy engine                                         |
|   - network scoper                                        |
+------------------------------+---------------------------+
                               |
                               v  exposes constrained surface (CLI / MCP / IPC)
+----------------------------------------------------------+
| Agent runtime                [UNTRUSTED]                  |
|   - LLM                                                   |
|   - tool dispatcher                                       |
|   - agent memory / RAG                                    |
+------------------------------+---------------------------+
                               |
                               v  reads from / writes to
+----------------------------------------------------------+
| External world               [UNTRUSTED]                  |
|   - RPC / Horizon endpoints                               |
|   - websites, emails, docs                                |
|   - peer agents, marketplaces                             |
+----------------------------------------------------------+
```

Two boundaries carry load:

- **Wallet core vs. agent runtime.** This boundary defines the attack surface. Every byte crossing it is adversarial input. The smaller and more constrained this surface, the cheaper the wallet is to harden.
- **User vs. wallet core.** The user establishes policy, unlocks the keystore, and approves high-value actions. The wallet must make this channel unambiguous (clear UI, signed approvals, no way for the agent to forge a "user said yes").

## 4. Threat catalog

Threats are IDed so they can be cited in the decision matrix and option docs. Likelihood and impact are ordinal (low / medium / high / critical). Defences listed are capability requirements, not design choices; the options doc maps them to concrete mechanisms per approach.

### T1. Key exfiltration from agent host
- **Description.** Malicious dependency, rogue process, or snapshot exposure copies the key material off the host.
- **Likelihood.** Medium (unattended hosts, long-running processes, large dependency graphs).
- **Impact.** Critical (full account loss on an unbounded key).
- **Defences required.** Keys in OS keyring / TPM / hardware-backed storage by default; never plaintext on disk; short-lived in-memory unlock windows; optional hardware-wallet signing; scoped subaccount keys so exfil of one key is not total loss; where the on-chain account is an OZ smart account, `External` signers with address-less public keys (EIP-7913-style verifier pattern) mean a stolen pubkey alone cannot produce a valid signature without the corresponding private key and verifier-compatible signing device, reducing exfil value at the protocol layer.
- **Sub-concern — browser-storage-adapter scope for web-companion SDK integrations.** If the companion surface adopts SAK's TypeScript SDK for a web build (`research/external/stellar-ecosystem/smart-account-kit.md`), the default storage adapters (`IndexedDBStorage` / `LocalStorageAdapter`) persist **credential metadata** — credential IDs plus contract-address associations — readable by any script in the origin. Private keys still live in the platform authenticator and are not at risk, but credential-to-contract linkage leakage is a targeting aid for social-engineering or phishing against the specific passkey. Defence: scope the origin tightly (no shared-subdomain hosting of other content), consider in-memory-only storage for ephemeral sessions, and document the exposure in the companion's privacy posture. KMP's storage adapters on native platforms (Keychain, Keystore) are not affected; web-target KMP inherits the same browser-storage exposure as SAK.

### T2. Prompt injection producing a malicious transaction
- **Description.** The agent reads attacker-controlled content (email, page, RAG doc, peer-agent reply) containing instructions that cause it to draft and submit a transaction favouring the attacker.
- **Likelihood.** High (this is the modal failure mode for agents transacting on-chain).
- **Impact.** High (bounded by policy if policy exists; critical otherwise).
- **Defences required.** Local policy engine enforcing per-tx, per-period, and per-counterparty limits; destination allowlists or identity-anchored allowlists; signed user-approved policy files; policy decisions logged with the draft they denied.

### T3. Hallucinated tool-call arguments
- **Description.** The agent constructs a syntactically valid but semantically wrong call (wrong account, wrong asset, wrong network, wrong amount unit).
- **Likelihood.** Medium-high (frequent in practice, often caught by structure but not always).
- **Impact.** Medium-high (amount-unit errors — stroops vs. XLM, 7-decimal shifts — are classic).
- **Defences required.** Schema-enforced typed parameters at the tool boundary (not free-form strings); explicit units in every amount field; network scoping that makes "wrong network" structurally impossible, not just unlikely; preview/dry-run that shows the resolved transaction in human terms before signing.

### T4. Delegated-authority abuse (sub-agent overreach)
- **Description.** A sub-agent granted scoped authority attempts to exceed it (more frequent, higher value, different counterparty, different asset).
- **Likelihood.** Medium.
- **Impact.** Bounded by policy if enforced; critical if the scope is advisory.
- **Defences required.** Delegation expressed in a form the wallet or the chain can enforce, not "share the key and hope." On-chain smart-account policy via OZ `stellar-accounts` context rules plus policies, Soroban auth entries with narrow scope, or locally enforced subaccount caps. Revocation that takes effect at the next `__check_auth` evaluation, not the next quarter — with the caveat that envelopes already signed and in flight can still succeed if submission races the revocation ledger, so `valid_until` windows on each rule must be bounded. See T11 (rule-ID downgrade), T12 (signer-set-divergence brick), T13-T14 (policy/verifier replacement) for the specific smart-account-layer sub-threats and their defences (auth-digest binding, atomic signer-and-threshold updates, wasm-hash pinning).

### T5. Counterparty impersonation
- **Description.** Lookalike domain, spoofed SEP-10 identity, memo-field phishing ("send to exchange with memo X"), or a marketplace entry claiming to be a known provider.
- **Likelihood.** Medium (common on existing chains; will rise with agents).
- **Impact.** High.
- **Defences required.** Identity-anchored allowlists (stellar.toml home-domain verified, known issuer) rather than address-only; explicit display of memo and federation-resolved identity; warnings for first-time counterparties under policy.

### T6. Runaway loop / economic exhaustion
- **Description.** A bug or manipulation causes the agent to submit many valid-looking small transactions until balance is exhausted or minimum reserves are breached.
- **Likelihood.** Medium.
- **Impact.** Medium (bounded by balance; can also render the account unusable by breaking reserves).
- **Defences required.** Rate limits and per-period caps at the policy layer; minimum-reserve guard; explicit "never below N XLM" rule that the wallet refuses to cross regardless of what the agent requests.

### T7. RPC tampering or inconsistent view
- **Description.** A compromised or malicious RPC endpoint returns misleading account state, fee estimates, or simulation results, causing the signer to approve a transaction under false premises.
- **Likelihood.** Low-medium (Horizon and Soroban RPC are typically trusted infrastructure, but agents may configure arbitrary endpoints).
- **Impact.** Medium-high.
- **Defences required.** Configurable multi-endpoint with cross-check for high-value actions; pinned TLS certs for user-operated endpoints; warnings when switching endpoints mid-session; display of which endpoint produced each piece of state.

### T8. Supply-chain compromise of wallet or plugins
- **Description.** A dependency, plugin, or "skill" contains malicious code that exfiltrates keys or rewrites transactions before signing.
- **Likelihood.** Low-medium (rising with plugin ecosystems).
- **Impact.** Critical.
- **Defences required.** Reproducible builds; signed releases; plugin/skill isolation (no access to the keystore, no ability to modify an already-authorised transaction); minimal default dependency set; auditable list of installed skills.
- **Sub-concern — hosted indexer dependency in client SDKs.** SAK defaults `rules.list()` / `rules.getAll()` to the hosted indexer at `smart-account-indexer.sdf-ecosystem.workers.dev` (testnet) / `smart-account-indexer-mainnet.sdf-ecosystem.workers.dev` (mainnet). KMP does the rule-enumeration scan on-chain natively (`OZContextRuleManager.getAllContextRules()`) and does not require the indexer for that feature, though it uses the indexer for credential-to-contract reverse lookup and multi-signer discovery when configured. The indexer reads only public on-chain event data, so **this is not a data-exposure vector** (corrects earlier draft framing). The residual concerns are a third-party liveness dependency if a wallet ships the default URL unchanged and a query-pattern side channel (the operator sees which credential IDs and accounts the wallet queries over time). Defence: never ship the hosted default on; either self-host, use KMP's on-chain scan for rule enumeration, use deterministic-address derivation for contract discovery, or accept the liveness dependency with eyes open (`REQ-sa-active-rule-enumeration`). See `research/external/stellar-ecosystem/smart-account-kit.md` §8, §10.

### T9. Social engineering via the approval channel
- **Description.** A high-value approval request is presented to the user in a way that hides the dangerous field (destination, memo, asset, amount). Habituation leads to auto-approval.
- **Likelihood.** Medium.
- **Impact.** High.
- **Defences required.** Approval UX that cannot be manipulated by the agent (wallet-controlled, not HTML-rendered agent output); highlighting of fields that differ from recent history; cooldowns on novel destinations; no way for the agent to fabricate an "approved" token.

### T10. Cross-session state confusion (testnet vs. mainnet)
- **Description.** The agent acts on mainnet while the user believed it was on testnet, or vice versa. Classic in developer workflows.
- **Likelihood.** Medium.
- **Impact.** High.
- **Defences required.** Network is a required, explicit parameter at every interface; no ambient "default network" that can be silently wrong; visible indicators in every response; separate keystores or separate namespaces per network.

### T11. Rule-ID downgrade by a malicious transaction sponsor
- **Description.** With OZ smart accounts (or any account following the OZ context-rule pattern), the `AuthPayload` carries a `context_rule_ids: Vec<u32>` selecting which stored rule governs each auth context. A malicious sponsor who collects legitimate signatures from a signer can, if the signer signed the raw host `signature_payload`, swap the `context_rule_ids` field at submission time to reference a more permissive rule, elevating the signer's effective authority without their consent.
- **Likelihood.** Low-medium (requires a compromised or malicious sponsor in the submission path).
- **Impact.** High (silently widens scope of signatures already granted).
- **Defences required.** Compute and sign the **auth digest** that binds rule IDs into the preimage: `sha256(signature_payload || context_rule_ids.to_xdr())`, per OZ `stellar-accounts` v0.7 hardening. The signed artefact MUST be this digest, not the raw `signature_payload`; off-chain signing flows that skip this step produce signatures that pass basic checks but fail `__check_auth` on a v0.7 or later account.

### T12. Signer-set-divergence rendering threshold policies unreachable
- **Description.** A context rule with an attached threshold policy (N-of-M) stores the threshold at install time. Removing signers from the rule without updating the threshold can leave the threshold permanently unreachable (a 5-of-5 rule with two signers removed becomes a 5-of-3 that cannot be satisfied). Adding signers without updating a weighted threshold silently weakens the security guarantee (a strict 3-of-3 becomes 3-of-5 after two additions, dropping the required approval from 100% to 60%).
- **Likelihood.** Medium (wallets that automate sub-agent provisioning will hit this through operational error).
- **Impact.** High (operational DoS on the rule; any funds gated by that rule become inaccessible without a recovery path).
- **Defences required.** Atomic signer-and-threshold updates via `ExecutionEntryPoint` in the same transaction; UI-level refusal to remove a signer when post-removal count drops below an attached threshold; audit-log event when a rule's signer set diverges from the stored threshold. This is a self-inflicted but non-obvious failure mode the OZ README flags explicitly under "Signer Set Divergence in Threshold Policies."

### T13. Policy or verifier contract replacement at deployment address
- **Description.** Context rules reference policy and verifier contracts by on-chain address. If the contract at that address is upgradeable or re-deployable, the rule's effective enforcement semantics can change without the account's explicit consent: a policy that enforced a spending limit could be upgraded to remove the check; a verifier that validated secp256r1 signatures could be upgraded to accept any signature. The account-side rule continues to point at the same address, with no built-in alarm.
- **Likelihood.** Low-medium (depends on how the referenced contracts are deployed; the OZ README recommends verifiers be immutable, but this is advisory).
- **Impact.** Critical (silently converts an enforced policy into no-op, or a scoped verifier into a universal accept).
- **Defences required.** Pin policy and verifier contracts by wasm hash at rule-install time; refuse install for mutable / upgradeable deployment addresses unless the user explicitly opts in; surface the pinned wasm hash in the approval UI; audit-log any detected wasm-hash change on a referenced contract. Complements T8 at the smart-account layer.

### T14. Cross-account verifier shared compromise
- **Description.** OZ verifier contracts are explicitly designed for sharing — one verifier validates signatures for many keys across many accounts (EIP-7913 pattern). A compromised or later-discovered-vulnerable verifier contract therefore affects every smart account that references it. Because verifiers are a recommended reuse primitive, the blast radius from a single verifier compromise is ecosystem-wide, not account-local.
- **Likelihood.** Low (well-known verifiers are audited; requires verifier compromise or latent bug disclosure).
- **Impact.** Critical (blast radius is every account referencing the compromised verifier).
- **Defences required.** Verifier pinning (T13 defence applies); preference for audited well-known immutable verifiers; where a wallet deploys many accounts, diversification across verifier implementations for high-value accounts; operational response plan for a disclosed verifier bug (rule-migration path to an alternative verifier).

## 5. Threat summary

| ID | Threat | Likelihood | Impact |
|---|---|---|---|
| T1 | Key exfiltration | Medium | Critical |
| T2 | Prompt-injected transaction | **High** | High |
| T3 | Hallucinated arguments | Medium-high | Medium-high |
| T4 | Delegated-authority abuse | Medium | Bounded-to-critical |
| T5 | Counterparty impersonation | Medium | High |
| T6 | Runaway loop | Medium | Medium |
| T7 | RPC tampering | Low-medium | Medium-high |
| T8 | Supply-chain compromise | Low-medium | Critical |
| T9 | Approval-channel manipulation | Medium | High |
| T10 | Network confusion | Medium | High |
| T11 | Rule-ID downgrade (malicious sponsor) | Low-medium | High |
| T12 | Signer-set-divergence threshold brick | Medium | High |
| T13 | Policy / verifier contract replacement | Low-medium | Critical |
| T14 | Cross-account verifier shared compromise | Low | Critical |

T2 is the highest-probability high-impact threat and the one that most differentiates this wallet from a conventional wallet. T1, T8, T13, and T14 are critical-impact threats with non-trivial likelihood that demand specific architectural defences. T3 and T10 are design-smell threats: they should be rendered *structurally impossible*, not merely unlikely. T11-T14 are smart-account-layer threats that only apply to wallets using OZ-style context rules (or equivalent); the wallet's defences at this layer are the v0.7 auth-digest rule-ID binding, wasm-hash pinning of policies and verifiers, and atomic signer / threshold-update discipline.

## 6. Architectural implications (teaser)

The analysis document treats these as requirements derived from the threat model rather than choices. Stated up front so the options doc can be judged against them:

1. **A local policy engine is mandatory**, not optional. The engine must sit inside the wallet core (the trust boundary), not in the agent runtime (the untrusted side). Any design that puts policy in the agent's prompt or in a sidecar the agent can bypass fails T2 and T4.
2. **The wallet surface to the agent must be typed and narrow.** Free-form transaction-XDR submission from the agent, while convenient, widens T3 considerably. The options doc discusses the trade-off.
3. **The approval channel must be wallet-owned.** Any design where the agent presents the "approve this" UI to the user fails T9.
4. **Keys are never the unit of delegation.** Delegation is expressed as policy (local or on-chain), not as key sharing. Follows from T1 and T4.
5. **Network identity is not optional state.** Every interface must require network selection explicitly and display it on every response. Follows from T10.

Whether these are best implemented CLI-first or agent-native-first is the subject of `06` and `07`. This document does not decide that; it fixes the bar both options must clear.
