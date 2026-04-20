# Fireblocks

**Status:** complete
**Researcher:** agent (tier-2 wave)
**Last updated:** 2026-04-18
**ID:** fireblocks
**Category:** crypto-agent

---

## 1. What it is

Fireblocks is an institutional digital-asset platform combining MPC-CMP key management, SGX/Nitro/Confidential-Space enclaves, and a Transaction Authorization Policy (TAP) engine. Customers operate vault accounts and whitelisted counterparties inside a Fireblocks-managed workspace; every signature is gated by an enclave-resident policy engine the customer configures but does not host [1][2][11]. The platform is the market reference for institutional custody governance — the policy vocabulary, approval workflows, and audit tooling are more mature than any tier-1 candidate we reviewed [11][12].

## 2. Agent interface

Programmatic access is via REST with official SDKs (TS, Python, Java, Go); SDK calls sign with an RSA key + JWT pair [6][8]. Minimum sequence for a signed action: (a) create an API user, pair with an API Co-signer, assign role (Signer/Admin/Approver/Editor) [5][6]; (b) whitelist destinations via Admin-Quorum approval [10]; (c) publish a TAP rule set (`POST` a rule list to the policy-editor endpoint; drafts can be validated before publish) [1][3]; (d) `POST /transactions`; the Fireblocks orchestrator routes the request to (a) AML screening, (b) TAP evaluation, (c) Admin-Quorum and/or designated approver workflow, (d) MPC signing across three key shares — two Fireblocks-held, one customer-held — and (e) broadcast [1][2][11]. There is no MCP / CLI / IDE surface.

## 3. Custody model

MPC-CMP with three key shares. Two shares live in Fireblocks-operated SGX enclaves on multiple clouds; the third is either a mobile signing device (warm/cold) or an API Co-signer the customer deploys in Intel SGX, AWS Nitro, or GCP Confidential Space — on cloud or on-premises [2][11]. The co-signer can be self-hosted, but the two Fireblocks-held shares and the policy/orchestration plane cannot — there is no fully-independent deployment [2]. Key Link allows a customer HSM/KMS to hold the signing key end-to-end instead of MPC, but still routes through the Fireblocks policy plane [12].

## 4. Policy and approval model

**TAP** is a top-down, first-match rule list. A matched rule's action is taken immediately; rules that match but do not explicitly approve fall through to the next rule; unmatched transactions default-deny [4][13]. Each rule combines an **initiator** (user or user-group), a **source** and **destination** resolved to workspace objects (VAULT, INTERNAL_WALLET, EXTERNAL_WALLET, CONTRACT, ONE_TIME_ADDRESS, NETWORK_CONNECTION, EXCHANGE, FIAT) [14], an **asset** scope, an **amount threshold** (per-transaction or periodic), a **transaction type** including `Contract call` decoded via verified ABI for per-function rules [13], and an **action**: Allow, Block, or X-of-Y approval via an `AuthorizationInfo` object listing groups (`AuthorizationGroup`) with per-group thresholds and an AND/OR combiner (`allowOperatorAsAuthorizer` controls whether initiator may also sign) [7].

**Admin Quorum** is orthogonal to TAP: a global n-of-m over all Admin/Owner/Non-Signing-Admin users that gates workspace changes — adding users, whitelisting addresses, approving network connections, editing policy itself [10]. **Approval Groups** let Admin Quorum duties be delegated per domain (security/compliance, user management, network, external accounts) with independent thresholds [10]. All policy is enforced server-side in the Fireblocks SGX plane and cannot be modified without admin-quorum approval [9][11]. Policy does not survive Fireblocks-operator compromise — it lives in their enclave, not the customer's.

## 5. Delegation model

Delegation is role- and group-based, not capability-based. An API user or human is assigned a role [5] and placed in user groups; TAP rules reference those groups as initiators/authorizers [1][7][10]. There is no bounded sub-key or scoped-signature primitive — revocation is a workspace-config change (edit user / edit rule / remove whitelist entry), gated by Admin Quorum, effective before the next TAP evaluation [10]. "Designated Signer" is the recommended pattern for third-party providers: give them Editor/Viewer only, force a Fireblocks-held signer into the approval path [6].

## 6. Relevant protocol coverage

Multi-chain custody including Stellar (XLM plus token activation) [8]. Enclave attestation on SGX/Nitro; MPC-CMP signing. No SEP-10 client, no x402. Rate limits and IP allowlisting per API key [6].

## 7. Licence and adoption signals

- Licence: proprietary service. Client SDKs Apache-2.0 (`fireblocks/ts-sdk`, `py-sdk`, `java-sdk`); `fireblocks/mpc-lib` is open [8].
- Source available: partial (SDKs + MPC primitives).
- Last meaningful commit: SDKs actively maintained (2025).
- Known production users: Revolut, BNY Mellon, banks/exchanges/funds with $10T+ cumulative transfers (marketing-claim tier; not independently verified).
- Commercial backing: Fireblocks Inc., late-stage private company.
- Ecosystem traction: SOC 2 Type 2, ISO 27001/27017/27018.

## 8. Non-negotiable check

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | no | Two of three MPC shares held by Fireblocks; policy plane held by Fireblocks. "Direct custody" branding notwithstanding, the customer cannot sign independently [2][11]. |
| N2 | Autonomous (no project-operated backend) | no | All signing coordinates through `api.fireblocks.io`; co-signer is one participant in a 3-party MPC run orchestrated by Fireblocks [2]. |
| N3 | No central server for keys/policy/history | no | Policy lives in Fireblocks SGX; transaction history and audit log are workspace-resident [1][11]. |
| N4 | Open source, permissive licence | no | Core platform proprietary. |
| N5 | JSON-default output | n/a | REST. |
| N6 | Testnet/mainnet parity | yes | Sandbox + Testnet + Mainnet workspace types, with documented feature deltas [15]. |

Fails N1/N2/N3/N4. Retained as a design reference, not a deployment model.

## 9. What to adopt

Compared to Coinbase CDP's typed criteria (`ethValue`/`evmAddress`/`evmData`/`netUSDChange`) and Turnkey's `{effect, consensus, condition}` triple, Fireblocks adds **governance as a first-class concept**, not just per-tx predicates. Concrete ideas expressible on a local self-custodial engine:

- **First-match top-down ordering with default-deny** [4][13]. Agents can reason about rule precedence deterministically; missing rule never means "allow." Addresses **T2**; serves **A1**, **A2**.
- **Typed source/destination object kinds beyond address** (`INTERNAL_WALLET` / `EXTERNAL_WALLET` / `CONTRACT` / `ONE_TIME_ADDRESS` / `NETWORK_CONNECTION`) [14]. Richer than Coinbase CDP's raw `evmAddress`; maps to Stellar as `{kind: contract | classic-account | federation-resolved | one-time, id: …}`. Addresses **T5** (identity-anchored rather than address-only); serves **A3**.
- **Contract-call rules keyed on decoded function selector + argument predicates** [13]. On Soroban this becomes a rule on `contract_id` + `function_name` + Soroban-auth-entry args. Direct **T2**/**T4** defence superior to opaque-data byte matching.
- **`AuthorizationInfo` approval object: list of groups, per-group threshold, AND/OR combiner, `allowOperatorAsAuthorizer` flag** [7]. On a local engine this becomes an approval-requirements object attached to a rule: `{groups: [{signers: [...], threshold: n}], logic: AND|OR, allowInitiatorAsApprover: false}`. Addresses **T4**, **T9**; serves **A2**, **A4**.
- **Admin Quorum as a separate governance plane** gating policy edits, allowlist changes, and user additions [10]. Translates to: policy-file itself requires n-of-m signatures to mutate, independent from transaction-signing keys. Addresses **T2**, **T8**.
- **Approval Groups scoped per workspace domain** (security, user-mgmt, network, external accounts) [10]. Maps to per-action-class quorums on our local engine — e.g. changing an allowlist requires different signers than adding a policy rule. Serves **A2**, **A7**.
- **Workspace roles as distinct authorities** (Admin, Non-Signing Admin, Signer, Approver, Editor, Viewer, Security Auditor) [5][6]. Wallet-side mapping onto A1-A7 (inferred from role semantics; not a Fireblocks-documented claim): A1 ~ Signer with narrow TAP; A2 ~ Admin (delegates) + per-child Approver groups; A3 ~ Signer with per-call scope suited to x402 / SEP-10 flows; A4 ~ Approver + Signer; A5 ~ narrow Signer; A6 ~ Viewer; A7 ~ Non-Signing Admin in the quorum.
- **Periodic-amount scope** alongside per-tx amount on every rule [4][13]. Addresses **T6** (runaway loop) natively rather than through a separate rate-limit layer.
- **Designated-Signer pattern**: grant third parties Editor/Viewer only; force a customer-held signer into the approval path [6]. On our engine: a rule type that forwards any matched transaction to a named local signer regardless of initiator. Addresses **T4**, **T8**.
- **Structured Audit Log with typed `EventId.*` categories** [16][17]. Every policy decision, every whitelist change, every admin-quorum approval emitted as a typed event. Our engine should emit `PolicyDecision`, `PolicyMutation`, `AllowlistMutation`, `ApprovalQuorumReached`, `SignerInvoked` as structurally distinct events. Addresses **T2**, **T4**, **T8**.
- **Contract-call decoded via verified ABI** before policy evaluation [13]. Analogue: decode Soroban auth entries to `(contract_id, function, args)` before policy match — never match on raw XDR.

## 10. What to avoid

- **Server-side enforcement of policy in vendor SGX** [1][2][11]. Fails N1/N2/N3. Our engine must evaluate locally in the wallet trust boundary.
- **MPC split where the vendor holds 2-of-3** [2][11]. Customer cannot sign or recover without Fireblocks. Fails N1/N2.
- **Whitelist mutation gated by vendor-side admin quorum** [10]. Adequate against external attackers; useless if the vendor is the adversary. On our engine, quorum must be verifiable locally, e.g. by multisig signatures on a policy file.
- **Role-and-group delegation as the only primitive** [5][10]. Group membership is a workspace-config edit, not a cryptographic capability — revocation requires Fireblocks availability. Fails **T4**'s "before next signature" bound when the vendor is unreachable.
- **No offline/break-glass signing path outside `fireblocks-key-recovery-tool` disaster recovery** [12]. Recovery is an emergency procedure, not an operating mode.
- **Opaque enclave attestation trust chain**. The customer is asked to trust Fireblocks-signed SGX code without independent binary attestation of the policy engine's current deployment.

## 11. Open questions

- Exact JSON schema and enum values of `PolicyRule` (type/action/srcType/dstType/amountScope/periodSec/operator/authorizers). Public docs point at `developers.fireblocks.com/reference/configure-transaction-authorization-policy` but the page did not render usable schema content over WebFetch [1]. Resolve via OpenAPI spec [8] before lifting the structure verbatim.
- Whether Fireblocks audit-log events are cryptographically chained / signed or only tamper-evident at the database layer [16][17].
- Whether TAP supports per-counterparty cumulative-spend caps natively or only per-destination aggregates [13].
- Whether Key Link deployments can enforce TAP client-side before the HSM signs, or whether policy still rides through the Fireblocks plane [12].

## 12. Sources

1. `https://developers.fireblocks.com/docs/set-transaction-authorization-policy` — accessed 2026-04-18; TAP overview and Policy-Editor API entry point.
2. `https://developers.fireblocks.com/docs/cosigner-architecture-overview` — accessed 2026-04-18; three-share split, enclave options, self-hosted boundaries.
3. `https://developers.fireblocks.com/reference/configure-transaction-authorization-policy` — accessed 2026-04-18; Policy-rule object reference (schema body not rendered).
4. `https://kevinsmall.dev/web3/fireblocks-101/` — accessed 2026-04-18; TAP initiator/from/to/asset/action model and top-down first-match semantics. Cited for mechanics only; not for normative claims.
5. `https://developers.fireblocks.com/reference/getvaultaccount` / `getpagedvaultaccounts` — accessed 2026-04-18; role enumeration on endpoint permissions (`Admin, Non-Signing Admin, Signer, Approver, Editor, Viewer`).
6. `https://developers.fireblocks.com/docs/manage-users`, `https://developers.fireblocks.com/docs/manage-api-keys`, `https://developers.fireblocks.com/docs/integrate-fireblocks-with-third-party-service-providers` — accessed 2026-04-18; role definitions, API-user constraints, Designated-Signer pattern.
7. `https://developers.fireblocks.com/reference/transaction-authorization-objects` — accessed 2026-04-18; `AuthorizationInfo` / `AuthorizationGroup` structure, `allowOperatorAsAuthorizer`, AND/OR logic.
8. `https://raw.githubusercontent.com/fireblocks/fireblocks-openapi-spec/main/open_api_spec.yml`, `https://github.com/fireblocks/ts-sdk`, `https://github.com/fireblocks/py-sdk`, `https://github.com/fireblocks/java-sdk`, `https://github.com/fireblocks/mpc-lib` — accessed 2026-04-18; OpenAPI spec and SDK surface; Stellar asset activation.
9. `https://www.fireblocks.com/principles` — accessed 2026-04-18; "policies encrypted and implemented within SGX… cannot be modified without admin quorum approval"; policy-rule categories (source/destination/asset/amount).
10. `https://developers.fireblocks.com/docs/define-approval-quorums` — accessed 2026-04-18; Admin Quorum and Approval Groups definitions, thresholds, domain scoping.
11. `https://developers.fireblocks.com/docs/capabilities`, `https://developers.fireblocks.com/docs/self-custody-infrastructure` — accessed 2026-04-18; hot/warm/cold wallet model, MPC-CMP, self-custody framing.
12. `https://docs.securosys.com/fireblocks/overview/`, `https://www.fireblocks.com/platforms/flexible-deployment/` — accessed 2026-04-18; Key Link HSM/KMS option; `fireblocks/fireblocks-key-recovery-tool` break-glass recovery.
13. `https://support.fireblocks.io/hc/en-us/articles/10293692115868-TAP-rules-for-specific-contract-call-methods` — accessed 2026-04-18; contract-call rules, ABI-decoded function matching, multicall handling.
14. `https://developers.fireblocks.com/reference/transaction-sources-destinations` — accessed 2026-04-18; source/destination enum (VAULT, EXCHANGE, INTERNAL_WALLET, EXTERNAL_WALLET, CONTRACT, ONE_TIME_ADDRESS, NETWORK_CONNECTION, FIAT).
15. `https://developers.fireblocks.com/docs/workspace-environments`, `https://developers.fireblocks.com/docs/get-to-know-fireblocks-workspaces` — accessed 2026-04-18; Sandbox/Testnet/Mainnet workspace types and role deltas.
16. `https://developers.fireblocks.com/reference/audit-log-events` — accessed 2026-04-18; typed `EventId.*` events across Administration/Policy/Transactions categories (index page).
17. `https://developers.fireblocks.com/reference/getauditlogs`, `https://developers.fireblocks.com/reference/getaudits` — accessed 2026-04-18; audit-log retrieval endpoints; `getaudits` deprecated in favour of `/management/audit_logs`.

## 13. Cross-links

- Compares to `coinbase-agentic.md` §4, §9 (typed-criteria policy schema) — Fireblocks adds object-typed source/dest and governance plane; criteria themselves less arithmetic.
- Compares to `turnkey.md` §4, §9 (`{effect, consensus, condition}` triple) — Fireblocks' `AuthorizationInfo` is the richer consensus expression (per-group thresholds, AND/OR, initiator-as-approver flag).
- Compares to `moonpay-agents-ows.md` §4 (local declarative rules + executable plugin) — Fireblocks shows the institutional ceiling of the declarative approach without local enforcement.
- Compares to `safe.md` §4, §5 (on-chain module-attachment + guards) — Safe is the self-custodial analogue; Fireblocks is the custodial-governance analogue. Our engine should combine Safe's enforcement locus with Fireblocks' vocabulary.
- Feeds requirement candidates: typed-destination-kind rule predicate; periodic-amount scope on every rule; `AuthorizationInfo`-shaped approval object; separate policy-mutation quorum; structured typed audit events; Designated-Signer rule action.
