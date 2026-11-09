> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Request a Quote

> Fetch solver pricing for cross-chain and same-chain intents using the LI.FI Intents order server.

The order server caches quotes from the solver network. Call `POST /api/v1/integrator/quote/request` to get pricing for an intent before opening an order.

Using the order server for quotes is optional but recommended. It matches intents against solver standing quotes and gives stronger execution guarantees.

No authentication is required. Optionally pass an `X-Integrator-Key` header: when set, the order server serves only solver quotes tagged for that key. It leaves out open-market quotes and does not fall back to them, so if no solver has tagged a quote for your key, you get none.

***

## Request Format

Every field that names an account or asset is a native address paired with a CAIP-2 chain id (`eip155:8453` for Base, `tron:728126428` for Tron, `solana:1151111081099710` for Solana mainnet). The example below quotes USDC from Base to Arbitrum (`exact-input`):

```bash curl theme={"system"}
curl -X POST 'https://order.li.fi/api/v1/integrator/quote/request' \
  -H 'Content-Type: application/json' \
  -d '{
    "user": {
      "chain": "eip155:8453",
      "address": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"
    },
    "intent": {
      "intentType": "oif-swap",
      "inputs": [{
        "chain": "eip155:8453",
        "user": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
        "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "amount": "10000000"
      }],
      "outputs": [{
        "chain": "eip155:42161",
        "receiver": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
        "asset": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
        "amount": null
      }],
      "swapType": "exact-input"
    },
    "supportedTypes": ["oif-user-open-v0"]
  }'
```

```ts TypeScript theme={"system"}
const response = await fetch('https://order.li.fi/api/v1/integrator/quote/request', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user: {
      chain: 'eip155:8453',
      address: '0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045',
    },
    intent: {
      intentType: 'oif-swap',
      inputs: [{
        chain: 'eip155:8453',
        user: '0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045',
        asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',  // USDC on Base
        amount: '10000000',  // 10 USDC (6 decimals)
      }],
      outputs: [{
        chain: 'eip155:42161',
        receiver: '0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045',
        asset: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831',  // USDC on Arbitrum
        amount: null,
      }],
      swapType: 'exact-input',
    },
    supportedTypes: ['oif-user-open-v0'],
  }),
});

const { quotes } = await response.json();
```

If no solver quote matches, the response is `200` with `"quotes": []`, not an error.

### Request Fields

| Field | Type | Description |
| - | - | - |
| `user` | object | `{ chain, address }` — CAIP-2 chain id plus the native address of the user (refund recipient on failure) |
| `intent.intentType` | string | Always `"oif-swap"` |
| `intent.inputs[]` | array | Input assets, each `{ chain, user, asset, amount }` with native addresses |
| `intent.outputs[]` | array | Desired outputs, each `{ chain, receiver, asset, amount }` (`amount: null` for exact-input) |
| `intent.swapType` | string | `"exact-input"` fixes the input amount and lets the solver determine the output (set `outputs[].amount` to `null`). This is the most common mode. `"exact-output"` fixes the output amount and lets the solver determine the required input (set `inputs[].amount` to `null`). Use this when the destination amount matters, for example to repay a loan or meet a contract requirement. |
| `supportedTypes` | array | Order types your integration supports. Pass `["oif-user-open-v0"]` — each quote returns an origin-chain transaction that opens the order. |
| `intent.metadata.exclusiveFor` | string or array | Optional. Restrict quote lookup to specific solver addresses. Useful for compliance or partnerships, but over-restriction can reduce fill reliability. |

The request shape is the same on every chain — the examples below only swap the `chain` values and address formats.

### Solana Request

Solana uses `solana:1151111081099710` (mainnet; devnet is `solana:1151111081099712`) and base58 addresses. A Solana-origin → EVM-destination example:

```bash curl theme={"system"}
curl -X POST 'https://order.li.fi/api/v1/integrator/quote/request' \
  -H 'Content-Type: application/json' \
  -d '{
    "user": {
      "chain": "solana:1151111081099710",
      "address": "8YLKoCu7NwqHNS8GzuvA2ibsvLrsg22YMfMDafxh1B15"
    },
    "intent": {
      "intentType": "oif-swap",
      "inputs": [{
        "chain": "solana:1151111081099710",
        "user": "8YLKoCu7NwqHNS8GzuvA2ibsvLrsg22YMfMDafxh1B15",
        "asset": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "amount": "1000000"
      }],
      "outputs": [{
        "chain": "eip155:8453",
        "receiver": "0x9773DAcbc46CAFb4e055060565e319922B48607D",
        "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "amount": null
      }],
      "swapType": "exact-input"
    },
    "supportedTypes": ["oif-user-open-v0"]
  }'
```

```ts TypeScript theme={"system"}
const response = await fetch('https://order.li.fi/api/v1/integrator/quote/request', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user: {
      chain: 'solana:1151111081099710',
      address: '8YLKoCu7NwqHNS8GzuvA2ibsvLrsg22YMfMDafxh1B15',
    },
    intent: {
      intentType: 'oif-swap',
      inputs: [{
        chain: 'solana:1151111081099710',
        user: '8YLKoCu7NwqHNS8GzuvA2ibsvLrsg22YMfMDafxh1B15',
        asset: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v', // USDC on Solana
        amount: '1000000',
      }],
      outputs: [{
        chain: 'eip155:8453',
        receiver: '0x9773DAcbc46CAFb4e055060565e319922B48607D',
        asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC on Base
        amount: null,
      }],
      swapType: 'exact-input',
    },
    supportedTypes: ['oif-user-open-v0'],
  }),
});

const { quotes } = await response.json();
```

The full list of Solana differences is in [Solana vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md).

Two Solana-specific rules about the request fields:

* **Use the `solana:` namespace for Solana legs.** A Solana leg quoted under `eip155:` is rejected — and the error message describes the asset's *encoding* (a hex-format or bytes32 complaint, depending on how the address was given), not the namespace mistake that actually caused it.
* **The receiver must be an on-curve wallet public key.** The API rejects PDAs and other off-curve addresses as `receiver` for Solana outputs.

### Tron Request

Tron uses `tron:728126428` and base58 `T…` addresses (`0x`-prefixed 20-byte hex is also accepted; responses always render base58). A Tron-origin → EVM-destination example:

```bash curl theme={"system"}
curl -X POST 'https://order.li.fi/api/v1/integrator/quote/request' \
  -H 'Content-Type: application/json' \
  -d '{
    "user": {
      "chain": "tron:728126428",
      "address": "TYYUrA6nV1EYxQqo7RiBJh6roFq4mtiBLT"
    },
    "intent": {
      "intentType": "oif-swap",
      "inputs": [{
        "chain": "tron:728126428",
        "user": "TYYUrA6nV1EYxQqo7RiBJh6roFq4mtiBLT",
        "asset": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "10000000"
      }],
      "outputs": [{
        "chain": "eip155:8453",
        "receiver": "0x841F63697cFa0e3B54c4D42b3d679F07F7F2485f",
        "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        "amount": null
      }],
      "swapType": "exact-input"
    },
    "supportedTypes": ["oif-user-open-v0"]
  }'
```

```ts TypeScript theme={"system"}
const response = await fetch('https://order.li.fi/api/v1/integrator/quote/request', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user: {
      chain: 'tron:728126428',
      address: 'TYYUrA6nV1EYxQqo7RiBJh6roFq4mtiBLT',
    },
    intent: {
      intentType: 'oif-swap',
      inputs: [{
        chain: 'tron:728126428',
        user: 'TYYUrA6nV1EYxQqo7RiBJh6roFq4mtiBLT',
        asset: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t', // USDT (TRC-20)
        amount: '10000000',
      }],
      outputs: [{
        chain: 'eip155:8453',
        receiver: '0x841F63697cFa0e3B54c4D42b3d679F07F7F2485f',
        asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913', // USDC on Base
        amount: null,
      }],
      swapType: 'exact-input',
    },
    supportedTypes: ['oif-user-open-v0'],
  }),
});

const { quotes } = await response.json();
```

Tron supports escrow settlement only — no Compact / resource-lock flow. Address conversion rules and the full delta list are in [Tron vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/tron-deltas.md).

***

## Response Format

```ts theme={"system"}
interface QuoteResponse {
  quotes: {
    order: {
      type: 'oif-user-open-v0';
      openIntentTx: OpenIntentTx;  // varies by origin chain — see below
      checks: {
        allowances: {
          chain: string;
          token: string;
          user: string;
          spender: string;
          required: string;
        }[];
      };
    };
    validUntil: number;
    quoteId: string;         // e.g. "quote_yCyE5aWW4NILo2UdM-8ETpia05TCLv"
    provider: 'LI.FI Intent';
    preview: {
      inputs: { chain: string; user: string; asset: string; amount: string }[];
      outputs: { chain: string; receiver: string; asset: string; amount: string }[];
    };
    metadata: {
      exclusiveFor: string | null;
    };
    partialFill: boolean;
    failureHandling: string; // e.g. "refund-automatic"
  }[];
}
```

The best quote is always at index 0.

### Response Fields

| Field | Description |
| - | - |
| `order.openIntentTx` | Ready-to-sign origin-chain transaction that opens the order. Its shape depends on the origin chain — see [The Open Transaction](#the-open-transaction). |
| `order.checks.allowances` | Allowance and balance checks that must hold before sending the transaction. Each item asserts the user holds and has approved at least `required` of `token` for `spender`. Always `[]` on Solana. |
| `validUntil` | Unix timestamp after which the quote expires. Re-fetch if stale. |
| `quoteId` | Reference ID that links your order to this quote for preferential solver matching. |
| `preview.inputs` / `preview.outputs` | The expected input and output amounts for this quote. Informational — verify against the order before signing. |
| `metadata.exclusiveFor` | The solver address selected for this quote, or `null`. |
| `partialFill` | Whether the solver supports partial fills for this quote. |
| `failureHandling` | How failures are handled (e.g. `"refund-automatic"`). |

### The Open Transaction

`order.openIntentTx` always carries `chain`, `to`, and `data`, plus one chain-specific fee field:

| Origin | `to` | `data` | Fee field |
| - | - | - | - |
| EVM | Input settler (checksummed hex) | Transaction calldata (hex) | `gasRequired` — gas estimate, decimal string |
| Solana | Input settler program id (base58) | Base58-serialized `VersionedTransaction` | `computeUnitsRequired` — compute-unit estimate, decimal string |
| Tron | Input settler (base58check) | Full ABI calldata (hex) — wrap in a `TriggerSmartContract` envelope | `feeLimit` — energy cap in SUN, decimal string |

Chain-specific handling before sending:

* **EVM**: satisfy every entry in `checks.allowances` (ERC-20 `approve` or Permit2) before broadcasting the calldata.
* **Solana**: the serialized transaction's `recentBlockhash` is a dummy all-zeros value (`11111111111111111111111111111111`) and **must be replaced with a fresh blockhash before signing**. The serialized message does not contain a Compute Budget instruction; rebuild it with a `ComputeBudgetProgram.setComputeUnitLimit` instruction using `computeUnitsRequired` before signing. `checks.allowances` is always `[]`: the SPL transfer happens inside the `open` instruction, so there is no approval pre-step.
* **Tron**: `data` is calldata, not a ready transaction — wrap it via TronWeb's `triggerSmartContract`, and approve the TRC-20 input first. `feeLimit` is a cap on energy spend, not an estimate.

The sign-and-send flow for each chain is in [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md).

### Exclusive Solver Matching

The `metadata.exclusiveFor` field identifies the solver selected to claim the inputs and therefore uses the **input chain's** native address format. When a quote wins, the order server encodes that identity and the exclusivity window into the built order's output context.

***

## Next Steps

> [card] [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md)
>
> Sign and send the quote's open transaction on each chain

> [card] [Track Order Status](https://docs.li.fi/lifi-intents/intents-api/track-status.md)
>
> Monitor order progress via the API or on-chain events


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.