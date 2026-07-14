---
name: masumi
description: >-
  Guides adding decentralized payments to AI agent services on Cardano using the
  Masumi protocol — MIP-003 agentic-service APIs, smart-contract escrow, on-chain
  agent registry, and decision logging. Triggers: "monetize my agent", "agent
  payments", "charge for my AI agent", "agent-to-agent payments", "MIP-003",
  "on-chain escrow for agents", "register agent on Cardano".
allowed-tools: Read Grep Glob
---

<!-- Documentation lookup path: ${CLAUDE_SKILL_DIR}/../../docs/sources/ -->

# Masumi Agent Payments

Help the developer connect an AI agent service to the Masumi protocol: an open-source,
self-hosted payment and registry layer on Cardano. Agents expose a standard HTTP API
(MIP-003), payments are held in a Cardano smart-contract escrow, delivered work is
committed on-chain as hashes (decision logging), and agents are discoverable through
an NFT-based on-chain registry.

## When to use

- Developer wants an AI agent (any framework: CrewAI, AutoGen, LangGraph, custom) to charge for its work
- Agent-to-agent (A2A) payments — one autonomous service hiring another without a human approving each transaction
- Trustless escrow between a service and unknown buyers — funds locked in a validator, released on delivery
- Making an agent service discoverable via an on-chain registry entry
- Verifiable delivery — proving what input produced what output without publishing the data (hash-based decision logging)
- Implementing or debugging a MIP-003 `start_job` / `status` service API

## When NOT to use

- Trusted parties only (internal company agents, known partners) — direct API calls and internal billing are simpler
- Micro-transactions well under ~1 ADA equivalent — per-transaction fees dominate; bundle into larger units instead
- Sub-second payment confirmation required — L1 settlement takes roughly one block (~20s); use a conventional payment processor
- General Cardano payment or transaction building without an agent-service context (use `build-transaction`)
- Querying chain data (use `query-chain`) or wallet integration in a dApp frontend (use `connect-wallet`)

## Key principles

1. **Blockchain must earn its place.** Escrow between mutually-distrusting parties, autonomous A2A payments, and portable on-chain reputation justify the complexity. If none of those apply, a conventional payment API is the better recommendation — say so.
2. **Everything is self-hosted and permissionless.** The payment service is a node the developer runs (Node.js + PostgreSQL), not a hosted dependency. Wallets are theirs; the protocol only defines the contracts and APIs.
3. **The service API and the payment layer are decoupled.** The agent implements MIP-003 (four small HTTP endpoints); the payment node handles chain interaction. Any language or framework that can serve HTTP works.
4. **Delivery is proven by hashes, not by trust.** sha256 hashes of canonicalized input and output are committed on-chain to unlock payment. Buyers recompute the hashes independently and request a refund on mismatch — get canonicalization exactly right (RFC 8785, UTF-8, semicolon delimiter).
5. **Preprod first, always.** The full flow — payment node, escrow lock, result submission, collection — should pass on the Preprod network with faucet funds before any mainnet key exists.
6. **Key hygiene is part of the integration.** Node-managed hot wallets hold operating funds only; collection goes to an external (ideally hardware) wallet configured by address, never by mnemonic.

## Workflow

### Step 1: Confirm the fit

Ask (if not already clear):

- **Who are the buyers?** Unknown third parties or other agents → escrow fits. Known/internal → recommend simpler billing and stop here.
- **Selling, buying, or both?** Seller needs the MIP-003 API + registry entry. Buyer needs discovery + purchase flow. Both is common for agent networks.
- **What stack is the agent in?** Python has an SDK (`pip-masumi`) that generates the MIP-003 endpoints; other languages implement four HTTP endpoints by hand.

### Step 2: Search bundled documentation

- `${CLAUDE_SKILL_DIR}/../../docs/sources/masumi/` — protocol documentation (payment service, registry, MIP specs)
- `${CLAUDE_SKILL_DIR}/../../docs/sources/cips/` — CIP-30/CIP-68 background for wallet and NFT metadata questions
- `${CLAUDE_SKILL_DIR}/references/mip-003-agentic-service-api.md` — endpoint specs, hashing rules, framework integration patterns
- `${CLAUDE_SKILL_DIR}/references/payment-service.md` — payment node setup, escrow flow, registry mint, refunds, troubleshooting

### Step 3: Stand up the payment node (Preprod)

The payment service is cloned and run locally: Node.js ≥ 18, PostgreSQL ≥ 14, a Blockfrost project key for Preprod, and three wallets — purchasing and selling (node-managed mnemonics) plus a collection wallet (external, address only). Setup commands, `.env` shape, and the admin dashboard are in `references/payment-service.md`. Fund the test wallets from the public Cardano testnet faucet before continuing.

### Step 4: Implement the MIP-003 service API (seller)

Four required endpoints on the agent service:

| Endpoint | Purpose |
|---|---|
| `POST /start_job` | Validate input, create a payment request, return the escrow address and identifier |
| `GET /status` | Report job state; on completion include output plus input/output hashes |
| `GET /availability` | Liveness — the registry checks this periodically |
| `GET /input_schema` | Machine-readable schema for `start_job` input |

The lifecycle: `start_job` creates a payment request against the payment node, the job runs only after the node observes funds locked on-chain, and completion submits `sha256(input) + sha256(output)` to unlock payment. Exact request/response bodies, status values, and per-framework skeletons (CrewAI, LangGraph, AutoGen) are in `references/mip-003-agentic-service-api.md`. In Python, `pip-masumi`'s `run()` generates all endpoints and the payment lifecycle.

### Step 5: Test the full escrow round-trip

Walk one job end-to-end on Preprod before anything else: payment request created → funds locked (visible on a Preprod explorer) → job executes → result hash submitted → dispute window passes → funds collected. The most common integration failures are hash mismatches from non-canonical JSON and wrong-network configuration — both are diagnosed in the references' troubleshooting tables.

### Step 6: Register the agent on-chain

Registration mints an NFT carrying the agent's metadata (name, description, API base URL, pricing, example outputs) into the selling wallet — this is what makes the service discoverable. Registration costs a small amount of ADA; querying the registry is free. Field names in the registry body are case-sensitive; the verified shape is in `references/payment-service.md`.

### Step 7: Buyer-side integration (if needed)

Buyers search the registry service for agents, call the advertised `start_job`, lock funds via their own payment node's purchase endpoint, poll `status`, then **independently recompute the hashes** before accepting the result — refund on mismatch. A complete TypeScript buyer flow is in `references/payment-service.md`.

### Step 8: Mainnet checklist

Only after Preprod passes cleanly:

- Separate mainnet API keys and wallets; hardware wallet as the collection target
- Realistic `submitResultTime` / dispute-window values (an hour or more, with buffer)
- Minimal float in node-managed hot wallets; auto-collection enabled
- Monitoring on `/availability` — an unreachable service is delisted from discovery and loses disputes

## References

- [mip-003-agentic-service-api.md](references/mip-003-agentic-service-api.md) — MIP-003 endpoint specifications, decision-log hashing (MIP-004), Python SDK fast path, framework patterns, testing
- [payment-service.md](references/payment-service.md) — payment node install and configuration, endpoint index with verified request bodies, seller/buyer flows, dispute and refund mechanics, fees, troubleshooting
- Protocol specs: https://github.com/masumi-network/masumi-improvement-proposals
- Payment service: https://github.com/masumi-network/masumi-payment-service
- Python SDK: https://github.com/masumi-network/pip-masumi
