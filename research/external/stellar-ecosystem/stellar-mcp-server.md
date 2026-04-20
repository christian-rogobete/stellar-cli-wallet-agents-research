# Stellar MCP server (christopherkarani/stellar-mcp)

**Status:** complete
**Researcher:** research-agent (Opus 4.7 1M)
**Last updated:** 2026-04-18
**ID:** stellar-mcp-server
**Category:** crypto-stellar

---

## 1. What it is

A single-file Node.js Model Context Protocol (MCP) server that exposes five Stellar tools over stdio to an MCP client (e.g. Claude Desktop). The repo cited by the RFP stub (`github.com/christopherkarani/stellar-mcp`) is a fork of `JoseCToscano/stellar-mcp` pinned at `44c42e4` (2025-03-31) [S1, S2]. It wraps `@stellar/stellar-sdk`, `passkey-kit`, and `sac-sdk` and submits Soroban transactions through Launchtube. There is no CLI, no policy engine, no allowlisting, no persistent state beyond two flat files (`agent-keys.txt`, `USAGE.md`).

## 2. Agent interface

MCP-only, stdio transport, five tools [S3]. The server class is constructed with `new McpServer({ name: 'stellar-mcp', version: '1.0.0', capabilities: { resources: {}, tools: {} } })` and `server.connect(new StdioServerTransport())` [S3, src/mcp-server.ts:25-32, 405]. There is no binary invocable on a shell other than `stellar-mcp-server` which only starts the MCP loop. Minimum signed-action sequence: (1) client reads the `Agent Keys` resource (plain-text file containing a Stellar secret) [S3, src/mcp-server.ts:38-49]; (2) client calls `sign-and-submit-transaction` with `transactionXdr`, `contractId`, `secretKey` all as free-form strings [S3, src/mcp-server.ts:222-235]; (3) server constructs `Keypair.fromSecret(secretKey)`, signs auth entries and envelope via `basicNodeSigner`, POSTs the XDR to Launchtube with a bearer JWT [S3, src/mcp-server.ts:247, 340-361; src/utils.ts:111-134].

Exact tool inventory: `create-account` (no args, returns `{publicKey, secretKey}` in plain text) [S3, src/mcp-server.ts:76-103], `fund-account(address)` (Friendbot GET) [S3, 105-142], `get-account(address)` (testnet Horizon only — URL hardcoded) [S3, 145-184], `get-transactions(address)` [S3, 186-220], `sign-and-submit-transaction(transactionXdr, contractId, secretKey)` [S3, 222-400].

## 3. Custody model

Keys live (a) in an environment-pointed plain-text file and (b) in arguments passed in every tool call by the agent. The `AGENT_KEYPAIR_FILE_PATH` env var points to a text file that the repo itself ships with a real-looking testnet secret (`agent-keys.txt`: `Secret Key: SCBRINS62LCZFTQJK3YDPYPS77YJH7LCVWKZMJFOJPOOIKO6SYP7GYR2`) [S4]. That file is registered as an MCP **resource** the agent can read over the protocol [S3, src/mcp-server.ts:38-49]. Separately the `sign-and-submit-transaction` tool requires the agent to pass `secretKey: z.string()` as a tool-call argument, which the server then feeds to `Keypair.fromSecret(secretKey)` in process memory [S3, src/mcp-server.ts:232-247]. Passkey mode uses the same pattern: the agent still supplies the classic `secretKey`, and passkey-kit signs *with* that keypair via `passkeyWallet.sign(transactionXdr, { keypair })` [S3, src/mcp-server.ts:288-292]. There is no OS keyring, no TPM, no hardware-wallet path, no unlock window. Launchtube and Mercury JWTs live in plain env vars [S5, src/utils.ts:32-39, 111-134].

## 4. Policy and approval model

None. There is no rate limit, no allowlist, no amount cap, no counterparty verification, no human approval channel, no dry-run mode. The server simulates the transaction only to branch read-vs-write [S3, src/mcp-server.ts:256-282]. Any policy enforcement is delegated to the remote smart wallet contract (via passkey-kit's policy signer flow [S3, src/mcp-server.ts:284-331; S6]); the MCP server itself is a signing oracle, not an enforcement point.

## 5. Delegation model

None at the MCP layer. Delegation is whatever the on-chain passkey smart wallet provides. The `shouldSignWithWalletSigner` helper detects C-address signers on an `AssembledTransaction.needsNonInvokerSigningBy()` and routes to passkey-kit [S3, src/utils.ts:47-74]. Revocation, scope, and bounds are entirely the smart-wallet contract's concern; this server does not know what it is authorising.

## 6. Relevant protocol coverage

- Classic payments: **no dedicated tool**. Submission is Soroban-only through `sign-and-submit-transaction` (requires a `contractId` starting with "C", 56 chars) [S3, src/mcp-server.ts:239-244].
- Soroban: yes, via pre-built XDR passed in from outside. The server does not build transactions itself.
- SEP-10: **not covered**. No challenge/response, no home-domain verification.
- SEP-24 / SEP-6 / SEP-12 / SEP-31: **not covered**.
- Asset contracts: `createSACClient` wraps `sac-sdk` [S3, src/utils.ts:148-158].
- Passkey: integrates `PasskeyKit` + `PasskeyServer` + Launchtube + Mercury [S3, src/utils.ts:18-39].
- x402: **not covered**.
- Network scoping: testnet is hardcoded in `get-account` and `get-transactions` via `https://horizon-testnet.stellar.org` [S3, src/mcp-server.ts:153-154, 196-197]; mainnet parity is not supported.

## 7. Licence and adoption signals

- Licence: ISC (fork file has no LICENSE file; `package.json` declares `"license": "ISC"`) [S7].
- Source available: yes.
- Last meaningful commit on the fork: 2025-03-31 (`44c42e4`, README edit) [S1]. Last code commit: 2025-03-28 (`Hanldes proper invalid auth error`) [S1].
- Fork stars: 0 / watchers 0 / forks 0 / contributors 1 (`JoseCToscano`) [S8]. Upstream `JoseCToscano/stellar-mcp` has 5 stars, 4 forks, 2 contributors (`Blockchain-Oracle` 108, `JoseCToscano` 17), and has since evolved into a larger code-generator project unrelated to the original simple MCP server [S9].
- Known production users: none found.
- Commercial backing: none. Solo developer.
- Ecosystem traction: Smithery listing and UBOS directory [S10, S11]; the simpler fork version is cited; upstream has diverged.

## 8. Non-negotiable check (§00-context §4)

| # | Non-negotiable | Result | Explanation |
|---|---|---|---|
| N1 | Self-custodial | partial | Keys stay on the host, but in plaintext and passed as tool arguments. T1/T2 exposure is severe. |
| N2 | Autonomous (no project-operated backend) | no | Hard dependency on Launchtube and Mercury hosted services [S3, src/utils.ts:32-39, 111-134]. |
| N3 | No central server for keys/policy/history | no | Launchtube mediates all submission; Mercury holds passkey wallet metadata. |
| N4 | Open source, permissive licence | yes | ISC [S7]. |
| N5 | JSON-default output | partial | Payloads are JSON-encoded but wrapped in MCP text content; no error envelope, free-form strings. |
| N6 | Testnet/mainnet parity | no | Horizon testnet URL hardcoded [S3, 153-154, 196-197]. |

Verdict: **fails N2, N3, N6; partial on N1, N5**. Not a deployment model; useful as a small prior-art illustration.

## 9. What to adopt

- **MCP resource pattern for read-only context**. Registering a plain-text usage guide and a key directory as named resources (`server.resource(...)`) is a clean way to provide agent-facing documentation inside the protocol rather than relying on the client's system prompt [S3, src/mcp-server.ts:38-74]. Applies to A3. Cite in `03-requirements.md` under discoverability.
- **Routing by signer shape**. `shouldSignWithWalletSigner` inspecting `needsNonInvokerSigningBy()` to choose between classic-keypair signing and smart-wallet passkey signing is a pattern we will need when supporting mixed G-accounts and C-accounts [S3, src/utils.ts:47-74]. Matches A2, A4.
- **Simulation-before-submit branch**. Using `simulate()` to detect read-only calls and short-circuit submission (saving fees) is a small but correct optimisation [S3, src/mcp-server.ts:256-282]. Counters part of T3 in read paths.

## 10. What to avoid

- **Secrets as tool arguments**. `secretKey: z.string()` accepted on every signing call means the agent's context window routinely carries raw `S...` secrets; an LLM log export or a compromised MCP client leaks them. Fails T1 and makes T8 catastrophic. The wallet must hold keys inside the trust boundary and never cross the boundary with them.
- **Plaintext key file as MCP resource**. Shipping `agent-keys.txt` with a real secret and exposing it as a readable resource means every agent prompt touches key material. Fails T1, T8. The repo even commits a functional testnet secret to the tree [S4] — a cultural tell that the custody model is not a goal.
- **No policy layer**. The server signs anything the agent asks it to sign, on any contract, to any counterparty, with no rate limit, allowlist, or amount cap. Fails T2 outright — this is the exact pattern `02-threat-model.md` §6(1) forbids.
- **Hardcoded network + no mainnet parity**. Horizon testnet baked into the URL strings [S3, src/mcp-server.ts:153-154, 196-197]. Fails T10 and N6; promoting to mainnet requires editing source.
- **Hard dependency on Launchtube and Mercury**. Every write path goes through a third-party bearer-token service [S3, src/utils.ts:111-134]. Fails N2, N3 and adds T7 surface.
- **Classic-payments gap**. No `payment` or `path-payment` tool; the only write surface is Soroban-XDR-in. A production agent wallet needs native classic ops.
- **No SEP-10 / SEP-24 / x402**. None of the SEPs or x402 that A3 and A4 require.
- **Single-user-single-host assumption**. The server is a stdio process configured with one env-var keypair; no notion of accounts, scopes, or tenants. Not a defect for its goal, but incompatible with A2's orchestrator pattern.

## 11. Open questions

- Does the upstream `JoseCToscano/stellar-mcp` direction (code-generator templates for Python/TypeScript MCP servers, as of 2026-03 commits [S9]) produce a server with different custody properties, or does it inherit the same "secret-as-tool-arg" pattern? Worth a separate Tier-2 look if the code-generator pattern becomes relevant.
- Is the passkey policy signer flow in passkey-kit rich enough to substitute for a local policy engine, or only sufficient for on-chain auth? Investigated in `research/external/stellar-ecosystem/passkey-kit.md`.

## 12. Sources

- [S1] Local clone [`stellar-mcp/`](https://github.com/christopherkarani/stellar-mcp) @ `44c42e4ad8db845e9d172e146dc3a486cbc6804f` (`https://github.com/christopherkarani/stellar-mcp`), accessed 2026-04-18.
- [S2] Upstream repo metadata via GitHub API `repos/JoseCToscano/stellar-mcp`, accessed 2026-04-18. Fork parent confirmed in `repos/christopherkarani/stellar-mcp` `parent` / `source` fields.
- [S3] [`stellar-mcp/src/mcp-server.ts`](https://github.com/christopherkarani/stellar-mcp/blob/main/src/mcp-server.ts) and `src/utils.ts` @ `44c42e4`.
- [S4] [`stellar-mcp/agent-keys.txt`](https://github.com/christopherkarani/stellar-mcp/blob/main/agent-keys.txt) @ `44c42e4` — real testnet secret committed to the tree.
- [S5] [`stellar-mcp/.env.example`](https://github.com/christopherkarani/stellar-mcp/blob/main/.env.example) @ `44c42e4`.
- [S6] [`stellar-mcp/README.md`](https://github.com/christopherkarani/stellar-mcp/blob/main/README.md) §Smart Contract Transaction Signing @ `44c42e4`.
- [S7] [`stellar-mcp/package.json`](https://github.com/christopherkarani/stellar-mcp/blob/main/package.json) `"license": "ISC"` @ `44c42e4`.
- [S8] GitHub API `repos/christopherkarani/stellar-mcp` and `repos/christopherkarani/stellar-mcp/contributors`, accessed 2026-04-18.
- [S9] GitHub API `repos/JoseCToscano/stellar-mcp/commits?per_page=30` and `/contributors`, accessed 2026-04-18 — shows divergence into a code-generator with Telegram bot, smart-wallet template, and Blockchain-Oracle as dominant contributor.
- [S10] Smithery listing `https://smithery.ai/server/@christopherkarani/stellar-mcp`, accessed 2026-04-18.
- [S11] UBOS listing `https://ubos.tech/mcp/stellar-blockchain-mcp-server/`, accessed 2026-04-18.

## 13. Cross-links

Directly compared to: `research/external/stellar-ecosystem/passkey-kit.md` (this server is a thin wrapper over passkey-kit's submission pipeline), `research/external/stellar-ecosystem/stellar-cli.md` (CLI-first counterexample), `research/external/crypto/kraken-cli.md` (MCP done as layer over a typed CLI, not as the sole surface). Feeds requirements for typed-parameter tool schemas (T3), local policy engine (T2), and key-material trust-boundary discipline (T1).

**Verdict for §09-decision-matrix: counter-example.** Not a basis to extend — the custody and policy stances are opposite to ours. Useful as a crisp illustration of what "MCP-only, secrets-as-arguments, no policy" produces, and why agent-native without a local policy engine is a non-starter under our threat frame.
