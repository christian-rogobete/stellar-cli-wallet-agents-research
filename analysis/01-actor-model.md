# Actor Model: Who Uses an AI-Agent Wallet

**Status:** draft — 2026-04-20
**Prerequisite for:** `03-requirements.md`, `04-axes.md`, `06`, `07`

---

## 1. Why start here

"AI-agent wallet" is ambiguous. A wallet optimised for an unattended subscription-paying daemon is a different artifact from a wallet used by a multi-agent orchestrator delegating budget to sub-agents, or from a user-facing assistant that surfaces high-value actions for human approval.

Conflating these produces generic designs that serve none of them well. The decision between CLI-first and agent-native is in large part a bet about **which actors dominate usage in the first two years**. This document defines the actors; the requirements doc maps needs onto them; the analysis doc scores each option against that map.

The catalog below is deliberately scoped to actors where the human is *not* continuously attending each signature — that is where the wallet's agent-specific value (policy engine, delegation primitives, session keys, wallet-owned approval) actually accrues. Developer-operator workflows — a human using Claude Code, Cursor, or a similar IDE agent to draft and ship Soroban contracts — are covered by actor A7 (human operator): from the wallet's perspective, a human reviewing every proposed command at a terminal is the same actor whether the proposal comes from their own keystrokes or from an IDE agent. The incumbent `stellar-cli` already serves that case well; the agent wallet inherits and polishes it rather than re-deriving it.

## 2. Classification dimensions

- **Attendedness** — is a human at a terminal during operation? (attended / semi-attended / unattended)
- **Transaction profile** — frequency and per-tx value distribution (sparse-and-small, sparse-and-large, frequent-and-small, frequent-and-variable)
- **Host reality** — where the process runs and the key lives (long-running VPS, ephemeral CI runner, user-owned device, orchestrator host, sub-agent container)
- **Delegation posture** — does this actor delegate spending authority to other actors? Does it receive delegated authority?
- **Acceptable blast radius** — worst-case damage from a single compromise, expressed as a fraction of the account

These dimensions are orthogonal in principle and all are used in the requirements doc. The actor list below foregrounds attendedness and host reality because those most strongly drive the centre-of-gravity decision.

## 3. Actor catalog

### A1. Unattended automation daemon

**Example.** A long-running process that pays a monthly subscription, tops up a Soroban contract balance, rebalances LP positions daily, or pays for indexer infrastructure from chain income.

**Host reality.** VPS, home server, or a container the user owns. Runs unattended for weeks or months. Keys on disk, ideally in an OS keyring or TPM-backed store.

**Transaction profile.** Frequent, small, bounded. Value-at-risk per period is known in advance.

**Delegation.** Receives a bounded budget from the human operator; does not re-delegate.

**Blast radius.** Should be bounded by policy to the allocated budget, regardless of agent behaviour. Host compromise is still total loss.

**Must-have capabilities.** Enforceable spending limits (per-tx, per-period, per-counterparty), auditable log of what was signed and why, graceful degradation when the policy denies (actionable error, not a crash), SEP-10/-45 client flows for paid services, x402 support, MPP support (charge mode and, when the formal channel spec lands, channel mode).

**Uncomfortable failure mode.** Prompt-injection via an RAG document or email causes the agent to draft a transaction that drains the budget to an attacker address. Policy must catch this without human intervention.

---

### A2. Multi-agent orchestrator with budget delegation

**Example.** A research assistant spawns sub-agents to price compute providers, buy data, and summarise results. Each sub-agent gets a budget; unspent budget returns to the orchestrator.

**Host reality.** Orchestrator host owns the root key. Sub-agents run in containers and get scoped signing authority, not the key itself.

**Transaction profile.** Frequent, small-to-medium, with strong per-sub-agent bounds.

**Delegation.** Heavy. This actor's primary pattern is scoped delegation to peers under its control.

**Blast radius.** Should be bounded per sub-agent. A compromised sub-agent loses at most its allocation.

**Must-have capabilities.** Subaccount creation (from mnemonic derivation and/or on-chain smart accounts). Fine-grained per-subaccount policy, implemented as **one context rule per sub-agent** rather than many signers on a shared rule — OZ's 15-signer hard cap per `ContextRule` makes per-sub-agent rules the correct architectural choice, and it preserves the isolation property that revoking one sub-agent affects only that sub-agent's rule. Revocation that takes effect at the `__check_auth` boundary before any *newly-constructed* signature can be produced (an envelope signed before revocation and still sitting in a mempool or retry queue can still succeed if submission races the revocation ledger — revocation is a forward-looking guarantee, not retroactive). Delegation expressible as a Soroban auth-entry or account-abstraction policy, not as "share the key." See `research/stellar-capabilities/04-smart-accounts.md` §3 and the OZ `stellar-accounts` README §"Use Cases / AI Agents" for the canonical template this actor maps onto.

**Uncomfortable failure mode.** Revocation lags a compromise: the orchestrator notices a rogue sub-agent but cannot stop a transaction whose auth entries were already signed and submitted before revocation. The wallet's defence is bounded `valid_until` windows on each sub-agent's rule so stale auth entries expire quickly on their own, plus a second defence of rate-limit and per-period caps on the policy attached to the rule.

---

### A3. Service-consumer agent (x402 / MPP / SEP-10/-45)

**Example.** An agent that buys API calls, compute minutes, or data on the user's behalf from a marketplace of service providers. Authentication via SEP-10 (G-account) or SEP-45 (C-account); payment via one of the HTTP-402-based agentic-payment protocols (x402 or MPP) or direct on-chain transfer.

**Host reality.** Usually the same host as its parent agent (A1); sometimes a dedicated process.

**Transaction profile.** Frequent, very small (cents to tens of cents), highly variable counterparties.

**Delegation.** Receives a budget scoped to service categories or trusted provider lists.

**Blast radius.** Per-counterparty caps are the main defence; allowlisting by category or SEP-10 identity is preferable to static destination allowlists.

**Must-have capabilities.** Native x402 flow (prepare-payment, pay, receipt); MPP charge and channel modes (see `research/stellar-capabilities/10-mpp.md`); SEP-10/-45 client including ephemeral keys for auth-only operations; receipts correlated with off-chain invoices.

**Uncomfortable failure mode.** A provider impersonates another provider via lookalike domains or metadata. The wallet must support identity-anchored policy (toml-verified home domain, known issuer), not just address-based allowlists.

---

### A4. User-facing assistant with human approval for high-value actions

**Example.** A personal assistant that handles small transfers automatically but surfaces anything above a threshold to the human for approval.

**Host reality.** User-owned device (phone companion app, laptop, or home assistant). Human is reachable but not always present.

**Transaction profile.** Mixed. Most transactions auto-approve under policy; a few require human ack.

**Delegation.** Receives broad but capped authority from the human.

**Blast radius.** Bounded by the auto-approve threshold while the human is available; bounded by per-period limits while they are not.

**Must-have capabilities.** Asynchronous approval channel (push, companion UI, WalletConnect-style), clear presentation of what will be signed, timeouts, graceful fallback when the human does not respond.

**Uncomfortable failure mode.** An approval request is approved on autopilot because the human is habituated and the presentation does not highlight the dangerous field (destination, memo, asset).

---

### A5. CI/CD and deployment agent

**Example.** A GitHub Actions workflow that deploys a contract, runs migrations, or publishes receipts. Keys from repo secrets or OIDC-issued short-lived credentials.

**Host reality.** Ephemeral runner. No persistent state. Key material injected at start of run.

**Transaction profile.** Infrequent, medium-to-high value per action, well-defined set of operations.

**Delegation.** Typically a dedicated deploy key with narrow authority.

**Blast radius.** Bounded by the scope of the deploy key. Secrets management discipline dominates wallet design here.

**Must-have capabilities.** Non-interactive unlock, environment-variable configuration, structured logs, dry-run mode, idempotent operations with explicit sequence-number handling.

**Uncomfortable failure mode.** A repo-wide secret leak exposes a key with broad authority. Scoped keys and on-chain policy reduce this risk; the wallet should make the scoped pattern easier than the permissive one.

---

### A6. Read-mostly research and indexer agent

**Example.** An agent that watches chain state, occasionally pays paid-indexer or paid-RPC vendor fees when deep history or high query throughput is needed, and otherwise does not transact. Chain-level fees are rare.

**Host reality.** Long-running service. Often does not need to hold significant balance; pays from a tiny topped-up hot account.

**Transaction profile.** Rare, tiny.

**Delegation.** Minimal.

**Blast radius.** Trivial by construction; keep the hot balance small.

**Must-have capabilities.** Clean separation of read operations (no key needed) from write operations (key needed). Read path must work without any signing setup.

**Uncomfortable failure mode.** The design forces a key on an actor that does not need one, broadening the threat surface for no reason.

---

### A7. Human operator (break-glass, debug, recovery, attended development)

**Example.** The human underneath any of the above actors. Sometimes inspects balances, sometimes recovers from a misbehaving agent, sometimes tests changes before letting the agent run. Also covers attended development — a developer using Claude Code, Cursor, or a similar IDE agent to draft and deploy Soroban contracts, where the agent proposes commands and the human approves each one at a terminal.

**Host reality.** Whatever machine the agent lives on, or a developer laptop during attended work.

**Transaction profile.** Ad-hoc during recovery; bursty (many testnet, few mainnet) during attended development.

**Delegation.** Root of trust.

**Blast radius.** Bounded by human attention; mainnet mistakes are the main risk (fat-finger deploys, wrong network) during attended sessions.

**Must-have capabilities.** Everything an agent can do, the human can do more directly and more safely. The human must never be worse off than the agent. Discoverability matters: `--help`, readable tables, sensible defaults. Strong network scoping (never "accidentally mainnet"), readable diff of what a transaction will do before signing, hardware-wallet support. First-class CLI with predictable JSON output; MCP transport for IDE-agent workflows.

**Uncomfortable failure mode.** The wallet is so agent-optimised that a human cannot use it to recover from an agent mistake without reading a protocol spec; or an IDE-agent session runs a mainnet command where the human expected testnet because the `--network` flag was omitted and the default was wrong.

---

## 4. Actor summary table

| ID | Actor | Attendedness | Tx profile | Key custody | Delegation | Lead requirement |
|---|---|---|---|---|---|---|
| A1 | Automation daemon | Unattended | Frequent, bounded | VPS/home server, disk-stored | Receives budget | Enforced local policy |
| A2 | Multi-agent orchestrator | Unattended | Frequent, per-child-bounded | Root key on orchestrator | Scoped delegation to sub-agents | Subaccounts + on-chain policy |
| A3 | Service-consumer agent | Semi-attended | Frequent, tiny, variable counterparty | Same as parent | Receives scoped budget | x402 / MPP / SEP-10/-45 identity policy |
| A4 | User-facing assistant | Semi-attended | Mixed | User device | Receives capped authority | Async human-approval channel |
| A5 | CI/CD deploy agent | Unattended | Infrequent, high-value | Ephemeral, env-injected | Narrow scope | Non-interactive, dry-run, idempotency |
| A6 | Read-mostly research | Unattended | Rare, trivial | Often none | None | Read path with no key required |
| A7 | Human operator (incl. attended dev) | Attended | Ad-hoc + bursty dev | Whatever is there; dev laptop | Root | Never worse off than the agent; network scoping; hardware signing |

## 5. Implications for the decision

Three observations from the catalog, without yet choosing an option:

1. **No single actor dominates.** A7 (human, including attended development) and A1 (unattended daemon) are at opposite ends of the attendedness axis and both are first-class. This argues against a design that optimises for one at the expense of the other.
2. **Delegation is the hard problem.** A2 and A4 both require authority to be granted to another actor with bounds that outlive the grant action. This needs either subaccounts, on-chain smart accounts with policy, or both. The CLI-vs-agent-native decision largely reduces to *where delegation lives*.
3. **Policy enforcement must be local.** N2/N3 (no central server) combined with A1's unattended profile means the policy engine lives on the user's host. The question is whether it is a CLI subsystem invoked by every command, a separate daemon, or a smart-contract boundary, not whether it exists.

These three points feed directly into `03-requirements.md` and `04-axes.md`.

## 6. Scoping note: actor consolidation (2026-04-20)

An earlier draft of this document listed a separate actor A1 "Developer-operator agent (attended)" covering developers using IDE agents like Claude Code or Cursor to draft and deploy Soroban contracts. That actor was consolidated into A7 (human operator) because, from the wallet's perspective, a human reviewing every proposed command at a terminal is the same actor whether the proposal originates from their own typing or from an IDE agent. Transport varies (terminal vs MCP over a local IDE bridge), but transport is not an actor-level property — the wallet still signs only what the human has reviewed, and the policy engine does not add value when the human is the policy. The consolidation sharpens the catalog to actors where the wallet's agent-specific features (policy engine, delegation, session keys, wallet-owned approval) actually accrue rather than actors who are structurally well-served by the incumbent `stellar-cli` plus a hardware wallet.
