> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Request a Quote

> Fetch solver pricing for cross-chain and same-chain intents using the LI.FI Intents order server.

The order server caches quotes from its solver network. Call `POST /quote/request` to get pricing for an intent before constructing an order.

Using the order server for quotes is optional but recommended. It matches intents against solver standing quotes and gives stronger execution guarantees.

<Note>
  For simpler integrator parameter handling, a V1 quote endpoint is also available at `GET /api/v1/integrator/quote/request`.
</Note>

***

## Request Format

<CodeGroup>
  ```bash curl theme={"system"}
  curl -X POST 'https://order.li.fi/quote/request' \
    -H 'Content-Type: application/json' \
    -d '{
      "user": "0x0001000002210514d8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
      "intent": {
        "intentType": "oif-swap",
        "inputs": [{
          "user": "0x0001000002210514d8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
          "asset": "0x0001000002210514833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
          "amount": "10000000"
        }],
        "outputs": [{
          "receiver": "0x0001000002A4B114d8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
          "asset": "0x0001000002A4B114af88d065e77c8cC2239327C5EDb3A432268e5831",
          "amount": null
        }],
        "swapType": "exact-input"
      },
      "supportedTypes": ["oif-escrow-v0"]
    }'
  ```

  ```ts TypeScript theme={"system"}
  const response = await fetch('https://order.li.fi/quote/request', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user: '0x0001000002210514d8dA6BF26964aF9D7eEd9e03E53415D37aA96045',
      intent: {
        intentType: 'oif-swap',
        inputs: [{
          user: '0x0001000002210514d8dA6BF26964aF9D7eEd9e03E53415D37aA96045',
          asset: '0x0001000002210514833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',  // USDC on Base
          amount: '10000000',  // 10 USDC (6 decimals)
        }],
        outputs: [{
          receiver: '0x0001000002A4B114d8dA6BF26964aF9D7eEd9e03E53415D37aA96045',
          asset: '0x0001000002A4B114af88d065e77c8cC2239327C5EDb3A432268e5831',  // USDC on Arbitrum
          amount: null,
        }],
        swapType: 'exact-input',
      },
      supportedTypes: ['oif-escrow-v0'],
    }),
  });

  const { quotes } = await response.json();
  ```
</CodeGroup>

### Request Fields

| Field                          | Type   | Description                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------ | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `user`                         | string | Interoperable address of the user (refund recipient on failure)                                                                                                                                                                                                                                                                                                                          |
| `intent.intentType`            | string | Always `"oif-swap"`                                                                                                                                                                                                                                                                                                                                                                      |
| `intent.inputs[]`              | array  | Input assets with `user`, `asset` (interoperable address), and `amount`                                                                                                                                                                                                                                                                                                                  |
| `intent.outputs[]`             | array  | Desired outputs with `receiver`, `asset`, and `amount` (`null` for exact-input)                                                                                                                                                                                                                                                                                                          |
| `intent.swapType`              | string | `"exact-input"` fixes the input amount and lets the solver determine the output (set `outputs[].amount` to `null`). This is the most common mode. `"exact-output"` fixes the output amount and lets the solver determine the required input (set `inputs[].amount` to `null`). Use this when the destination amount matters, for example to repay a loan or meet a contract requirement. |
| `supportedTypes`               | array  | Resource lock types the user supports. Use `"oif-escrow-v0"` for standard escrow integrations; include `"oif-resource-lock-v0"` only if your flow supports Compact/resource-lock claims                                                                                                                                                                                                  |
| `intent.metadata.exclusiveFor` | array  | Optional. Restrict quotes to specific solver addresses. Useful for compliance or partnerships, but over-restriction can reduce fill reliability.                                                                                                                                                                                                                                         |

***

## Response Format

```ts theme={"system"}
interface QuoteResponse {
  quotes: {
    order: null;
    validUntil: number;
    quoteId: string;         // e.g. "quote_yCyE5aWW4NILo2UdM-8ETpia05TCLv"
    preview: {
      inputs: { user: string; asset: string; amount: string }[];
      outputs: { receiver: string; asset: string; amount: string }[];
    };
    metadata: {
      exclusiveFor: string | null;
    };
    partialFill: boolean;
    failureHandling: string; // e.g. "refund-automatic"
  }[];
}
```

The best quote is always at index 0. Save the `quoteId` for use when [submitting the order](/lifi-intents/intents-api/create-and-submit).

### Response Fields

| Field                                | Description                                                                        |
| ------------------------------------ | ---------------------------------------------------------------------------------- |
| `validUntil`                         | Unix timestamp after which the quote expires. Re-fetch if stale.                   |
| `quoteId`                            | Reference ID to include when submitting an order for preferential solver matching. |
| `preview.inputs` / `preview.outputs` | The expected input and output amounts for this quote.                              |
| `metadata.exclusiveFor`              | The solver address selected for this quote, or `null`.                             |
| `partialFill`                        | Whether the solver supports partial fills for this quote.                          |
| `failureHandling`                    | How failures are handled (e.g. `"refund-automatic"`).                              |

### Exclusive Solver Matching

The `metadata.exclusiveFor` field in the response identifies the solver selected for this quote. If you want to enforce exclusivity when constructing the order, encode the solver address and an expiry into the output's `context` field:

```ts theme={"system"}
const exclusiveFor = bestQuote.metadata.exclusiveFor;
if (exclusiveFor) {
  const currentTime = Math.floor(Date.now() / 1000);
  const exclusiveExpiry = (currentTime + 60).toString(16);
  const paddedExclusiveFor = exclusiveFor.replace('0x', '').padStart(64, '0');
  mandateOutput.context = `0xe0${paddedExclusiveFor}${exclusiveExpiry}`;
}
```

Apply exclusivity only if you fully control solver policy and fallback behavior.

***

## Next Steps

<CardGroup cols={2}>
  <Card title="Create and Submit Orders" icon="paper-plane" href="/lifi-intents/intents-api/create-and-submit">
    Construct a StandardOrder and submit it to the solver network
  </Card>

  <Card title="Track Order Status" icon="signal" href="/lifi-intents/intents-api/track-status">
    Monitor order progress via the API or on-chain events
  </Card>
</CardGroup>
