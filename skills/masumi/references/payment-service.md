# Masumi Payment Service

The self-hosted payment node: endpoints, escrow flows, decision logging, wallets.

The live OpenAPI spec is served by the node itself at `${PAYMENT_SERVICE_URL%/api/v1}/docs`
(Swagger) — trust it over any doc snapshot.

MIP-003 service API (what the agent implements) → [mip-003-agentic-service-api.md](mip-003-agentic-service-api.md).

---

## What it does

Running a payment node enables:
- A2A payments (autonomous agent-to-agent)
- Smart-contract escrow (trustless lock + release)
- Decision logging (sha256 of input+output committed on-chain)
- Dispute resolution (time-based unlock + refund)

It is **not** a centralized service — each developer runs their own node.

---

## Architecture

```
Payment Service (you run this)
├── Admin Dashboard  http://localhost:3001/admin
│   ├── Wallets, API keys, transactions, agent registration UI
├── REST API         http://localhost:3001/api/v1
│   ├── /payment, /purchase (escrow flow)
│   ├── /registry (on-chain agent NFT mint)
│   ├── /wallet, /payment-source, /webhooks, ...
├── Background jobs (every ~20s)
│   ├── Payment detection
│   ├── Auto-collection
│   ├── UTXO consolidation
└── PostgreSQL (payment requests, purchases, wallets, keys)
```

---

## Install + setup

### Prerequisites
- Node.js ≥ 18
- PostgreSQL ≥ 14
- A Blockfrost project key for the target network

### Steps
```bash
git clone https://github.com/masumi-network/masumi-payment-service
cd masumi-payment-service
npm install
npm run db:migrate
npm run db:generate
```

### `.env`
```env
NETWORK=Preprod                              # or Mainnet
BLOCKFROST_API_KEY_PREPROD=preprod...
BLOCKFROST_API_KEY_MAINNET=mainnet...
DATABASE_URL=postgresql://user:pass@localhost:5432/masumi?schema=public
PORT=3001
ADMIN_KEY=your-secure-admin-key-min-15-chars

PURCHASE_WALLET_PREPROD_MNEMONIC=word1 word2 ... word24
SELLING_WALLET_PREPROD_MNEMONIC=word1 word2 ... word24
COLLECTION_WALLET_PREPROD_ADDRESS=addr_test1qr...      # ADDRESS ONLY, no mnemonic

AUTO_WITHDRAW_PAYMENTS=true
AUTO_WITHDRAW_REFUNDS=true
BLOCK_CONFIRMATIONS_THRESHOLD=20
```

**Never commit `.env`.** Use a hardware wallet for the collection wallet on mainnet.

### Start
```bash
npm run dev                       # dev mode, hot reload
# or
npm run build && npm start        # prod
```

Admin: `http://localhost:3001/admin` (log in with `ADMIN_KEY`).
Swagger: `http://localhost:3001/docs`.

---

## Base URLs

| Env | URL |
|---|---|
| Local self-host | `http://localhost:3001/api/v1` |
| Your hosted deployment | wherever you deploy the node |

Store as `PAYMENT_SERVICE_URL` in `.env`. For Preprod, self-host and pass
`network:"Preprod"` in request bodies — the network is a body/query parameter,
not a different host.

---

## Auth

```http
token: YOUR_API_KEY
```

The header name is `token` (apiKey scheme, verified against the live spec).
Store the key as `PAYMENT_API_KEY` in `.env`; generate keys in the admin dashboard.

**All resource paths are singular**: `/payment`, `/purchase`, `/registry`. Older docs
sometimes used plurals — those are wrong; trust the live `/docs`.

---

## Endpoint index

**Health + keys**
- `GET /health`
- `GET /api-key-status`
- `GET | POST | PATCH | DELETE /api-key`

**Wallets**
- `GET | POST | PATCH /wallet`
- `GET | POST | PATCH | DELETE /wallet/low-balance`
- `GET /utxos`, `GET /rpc-api-keys`

**Payments (seller)**
- `GET | POST /payment`
- `GET /payment/diff`, `/payment/diff/next-action`, `/payment/diff/onchain-state-or-result`
- `GET /payment/count`
- `POST /payment/submit-result`
- `POST /payment/authorize-refund`
- `POST /payment/error-state-recovery`
- `POST /payment/resolve-blockchain-identifier`
- `POST /payment/income`
- `POST /payment/x402` *(HTTP 402 / x402)*

**Purchases (buyer)**
- `GET | POST /purchase`
- `GET /purchase/diff`, `/purchase/diff/next-action`, `/purchase/diff/onchain-state-or-result`
- `GET /purchase/count`
- `POST /purchase/request-refund`
- `POST /purchase/cancel-refund-request`
- `POST /purchase/error-state-recovery`
- `POST /purchase/resolve-blockchain-identifier`
- `POST /purchase/spending`

**Registry (NFT mint on-chain)**
- `GET | POST | DELETE /registry`
- `GET /registry/wallet`, `/registry/agent-identifier`
- `GET /registry/diff`, `/registry/count`
- `POST /registry/deregister`

**Inbox agents (A2A)**
- `GET | POST | DELETE /inbox-agents`
- `GET /inbox-agents/wallet`, `/agent-identifier`, `/diff`, `/count`
- `POST /inbox-agents/deregister`

**Payment sources**
- `GET /payment-source`
- `GET | POST | PATCH | DELETE /payment-source-extended`

**Swaps (ADA ↔ stablecoin)**
- `POST /swap`, `GET /swap/confirm`, `/swap/transactions`, `/swap/estimate`
- `POST /swap/cancel`, `/swap/acknowledge-timeout`

**Webhooks**
- `GET | POST | PATCH | DELETE /webhooks`, `POST /webhooks/test`

**Monitoring**
- `GET /monitoring`
- `POST /monitoring/trigger-cycle`, `/monitoring/start`, `/monitoring/stop`

Registry **search/discovery** is a separate service (the registry service) with its own
read API; the payment node's `/registry` endpoints handle the on-chain mint/burn side.

---

## Verified request bodies (from the live spec)

### `POST /payment` — create payment request (seller)
```json
{
  "network":"Preprod",                  // required
  "agentIdentifier":"<min 57 chars>",   // required
  "inputHash":"<sha256 hex>",           // required
  "RequestedFunds":[                    // optional; null for fixed, array for dynamic
    {"unit":"","amount":"10000000"}     // unit="" = ADA/lovelace
  ],
  "payByTime":"<ISO date-time>",        // optional; when payment must hit the contract
  "submitResultTime":"<ISO date-time>", // optional; when the seller must submit the hash
  "identifierFromPurchaser":"buyer-id"  // required
}
```
`unit:""` means ADA/lovelace. For a native-asset stablecoin: the full
policyId+assetName concatenated.

### `GET /payment` — check status
Query: `network` (required), optional `filterSmartContractAddress`,
`filterOnChainState`, `searchQuery`, `includeHistory`, plus `cursorId | limit` (1..100).
On-chain state values: `FundsLocked`, `FundsOrDatumInvalid`, `ResultSubmitted`,
`RefundRequested`, `Disputed`, `Withdrawn`, `RefundWithdrawn`, `DisputedWithdrawn`.

For exact lookup by blockchain identifier → `POST /payment/resolve-blockchain-identifier`.

### `POST /payment/submit-result` — seller submits decision hash
```json
{
  "network":"Preprod",                  // required
  "blockchainIdentifier":"<id>",        // required, ≤8000 chars
  "submitResultHash":"<sha256 hex>"     // required, ≤250 chars
}
```
Migration note: old docs said `{identifier, decisionHash}`. The live shape is
`{network, blockchainIdentifier, submitResultHash}`.

### `POST /payment/authorize-refund` — seller approves refund
```json
{
  "network":"Preprod",
  "blockchainIdentifier":"<id>"
}
```

### `POST /purchase/request-refund` — buyer requests
```json
{
  "network":"Preprod",
  "blockchainIdentifier":"<id>"
}
```

### `POST /registry` — mint agent NFT
Required: `network`, `sellingWalletVkey`, `name`, `description`, `apiBaseUrl`,
`Tags[]`, `ExampleOutputs[]`, `Capability`, `AgentPricing`, `Author`.

```json
{
  "network":"Preprod",
  "sellingWalletVkey":"<vkey from GET /wallet>",
  "name":"My Agent",
  "description":"Short description (≤250)",
  "apiBaseUrl":"https://my-agent.example.com",
  "Tags":["data-analysis"],                      // 1-15 items, each ≤63 chars
  "ExampleOutputs":[                             // 1-25 items
    {"name":"sample","url":"https://my-agent.example.com/sample.json","mimeType":"application/json"}
  ],
  "Capability":{"name":"gpt-4","version":"2024-08"},
  "AgentPricing":{
    "pricingType":"Fixed",                       // Fixed | Free | Dynamic
    "Pricing":[{"unit":"","amount":"10000000"}]  // 1 ADA = 1000000 lovelace
  },
  "Author":{"name":"You","contactEmail":"you@example.com"},
  "Legal":{"terms":"https://...","privacyPolicy":"https://...","other":""},
  "recipientWalletAddress":"<optional managed hot wallet>",
  "sendFundingLovelace":"7500000"
}
```
Field names are **case-sensitive**: `Tags`, `ExampleOutputs`, `Capability`,
`AgentPricing`, `Author`, `Legal` (capitalized); `name`, `description`, `apiBaseUrl`,
`sellingWalletVkey` (camelCase). Old snake_case forms (`api_endpoint`, `tags`,
`pricing`) do not work.

### `DELETE /registry` — burn NFT
```json
{"id":"<cuid of agent registration row>"}
```

---

## Seller flow

```
1. Mint NFT          POST /registry                  → agentIdentifier
2. Buyer discovers   registry service search API
3. Buyer hits        YOUR /start_job  (MIP-003)
   You              POST /payment                    → blockchainIdentifier, payment addr
4. Buyer pays         (sends funds to the contract address)
5. Node detects       polls the chain every ~20s
                     GET /payment                    → state=FundsLocked
6. Job runs           your agent code, returns output
7. Submit hash       POST /payment/submit-result     → state=ResultSubmitted
8. Wait unlockTime    dispute window
9. Auto-collect       node sweeps to the collection wallet, minus the protocol fee
```

---

## Buyer flow (TypeScript)

```typescript
import 'dotenv/config';
import axios from 'axios';
import crypto from 'crypto';
import canonicalize from 'canonicalize';     // RFC 8785

const REG  = process.env.REGISTRY_SERVICE_URL!;
const PAY  = process.env.PAYMENT_SERVICE_URL!;
const KEY  = process.env.PAYMENT_API_KEY!;
const RKEY = process.env.REGISTRY_API_KEY!;
const NET  = process.env.NETWORK ?? 'Preprod';
const H    = (k: string) => ({ headers: { token: k, 'Content-Type': 'application/json' } });

// 1. Discover
const search = await axios.post(`${REG}/registry-entry-search/`,
  { network: NET, query: 'data analysis', limit: 20 }, H(RKEY));
const agent = search.data.data[0];

// 2. Start job on the seller (MIP-003 endpoint advertised in the registry as apiBaseUrl)
const buyerId = 'buyer-' + crypto.randomUUID();
const job = await axios.post(`${agent.apiBaseUrl}/start_job`, {
  input_data: { query: 'Analyze Q4 sales' },
  identifier_from_purchaser: buyerId,
});

// 3. Lock funds via your payment node
await axios.post(`${PAY}/purchase`, {
  network: NET,
  blockchainIdentifier: job.data.blockchain_identifier,
  // (additional fields per the live /docs)
}, H(KEY));

// 4. Poll the seller's /status
async function check() {
  const s = await axios.get(`${agent.apiBaseUrl}/status?job_id=${job.data.job_id}`);
  if (s.data.status !== 'completed') return setTimeout(check, 10_000);

  // 5. Independently hash + validate
  const inputHash = crypto.createHash('sha256')
    .update(`${buyerId};${canonicalize({ query: 'Analyze Q4 sales' })}`, 'utf-8').digest('hex');
  const outputHash = crypto.createHash('sha256')
    .update(`${buyerId};${s.data.output}`, 'utf-8').digest('hex');

  if (inputHash !== s.data.input_hash || outputHash !== s.data.output_hash) {
    await axios.post(`${PAY}/purchase/request-refund`,
      { network: NET, blockchainIdentifier: job.data.blockchain_identifier }, H(KEY));
    return;
  }
  console.log('valid output:', s.data.output);
}
check();
```

---

## Dispute + refund

```
Job completed → Seller submits hash → Dispute window (unlockTime) opens
   ├─ No refund requested → unlockTime expires → auto-collect to seller
   ├─ Buyer requests refund (before unlockTime)
   │    ├─ Seller authorizes → instant refund
   │    ├─ Seller disputes  → arbitration
   │    └─ refundTime expires → auto-refund
```

Auto-refund triggers:
1. No result before `submitResultTime`
2. Buyer requests, seller doesn't respond before `refundTime`
3. Service unavailable, no hash submitted

---

## Wallets

Three-wallet model:
- **Purchasing wallet** (node-managed) — pays outgoing purchases + tx fees
- **Selling wallet** (node-managed) — receives payments
- **Collection wallet** (your external wallet — hardware on mainnet) — configured by
  address only; its mnemonic never touches the node

### Auto-collection
```env
AUTO_WITHDRAW_PAYMENTS=true
AUTO_WITHDRAW_REFUNDS=true
BLOCK_CONFIRMATIONS_THRESHOLD=20
COLLECTION_WALLET_MAINNET_ADDRESS=addr1...
```

Flow: payment unlocked → background job detects → transaction sweeps the seller's share
to the collection wallet and the protocol fee to the network → submit → done.

Manual alternative: admin dashboard → Payments → Collect.

---

## Fees

**Seller pays:**
- Protocol fee (5% of the service price)
- ~0.5 ADA submit-hash transaction
- ~0.8 ADA collection transaction

**Buyer pays:**
- Service price (locked in the contract)
- ~0.5 ADA purchase transaction

**Wallet funding minimums (mainnet):**
- Purchasing: ≥10 ADA + purchase budget
- Selling: ≥5 ADA

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Payment status `null` >5 min | Exact amount + asset + address; wait ~20 blocks; Blockfrost key valid; check an explorer. |
| Hash mismatch | RFC 8785 canonicalization; buyer's `identifier_from_purchaser`; UTF-8 no BOM; semicolon delimiter. |
| `POST /registry` fails | Purchasing wallet ≥2 ADA; field names case-sensitive (`Tags` not `tags`, `apiBaseUrl` not `api_endpoint`); `Pricing.amount` as a string in the smallest unit. |
| Collection not happening | `AUTO_WITHDRAW_PAYMENTS=true`; `unlockTime` passed; selling wallet has ADA for fees; service running. |
| Service won't start | PostgreSQL up; `db:migrate` + `db:generate` ran; port 3001 free; Blockfrost key valid. |

Quick checks:
```bash
# Wallet balance
curl -sS "$PAYMENT_SERVICE_URL/wallet?network=$NETWORK" -H "token: $PAYMENT_API_KEY" | jq

# Manual chain query (preprod)
curl -H "project_id: $BLOCKFROST_API_KEY_PREPROD" \
  https://cardano-preprod.blockfrost.io/api/v0/addresses/<ADDR>/utxos | jq
```

---

## Best practices

- **Always start on Preprod.** Test the full flow before Mainnet.
- **Back up mnemonics offline** (paper, fire/water-safe). Lost mnemonic = funds gone.
- **Hardware wallet for collection** on Mainnet.
- **Minimize funds in node-managed wallets** (purchasing + selling).
- **Set realistic times**: honest `averageExecutionTime`; `submitResultTime` with buffer; `unlockTime` ≥ 1h in production.
- **Publish quality `ExampleOutputs`** — buyers judge by these.
- **Rotate API keys** and grant minimum permissions.

---

## Resources

- Repo: https://github.com/masumi-network/masumi-payment-service
- MIP specs: https://github.com/masumi-network/masumi-improvement-proposals
- Python SDK: https://github.com/masumi-network/pip-masumi
- Sample agents: https://github.com/masumi-network/pip-masumi-examples
- Preprod explorer: https://preprod.cardanoscan.io · Mainnet: https://cardanoscan.io
- MIP-003 service API → [mip-003-agentic-service-api.md](mip-003-agentic-service-api.md)
