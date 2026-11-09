> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Compact Orders

> Submit gasless off-chain orders via The Compact resource lock. Deposit once, issue multiple intents without per-order gas costs.

The Compact flow uses [The Compact](https://github.com/Uniswap/the-compact) resource lock for off-chain, gasless order submission. Users deposit tokens once and can issue multiple intents from that balance without paying gas per order.

> [!NOTE]
>
> If you're adding LI.FI Intents alongside other bridges, the [standard escrow flow](https://docs.li.fi/lifi-intents/quickstart.md) is simpler and recommended. Use the Compact flow when you need gasless submissions or are building a resource-lock-native application.

***

## How It Differs from Escrow

| | Escrow | Compact |
| - | - | - |
| Token locking | Per-intent, on-chain each time | Deposit once, reuse balance |
| Order submission | On-chain (`open`/`openFor`) | Off-chain (`POST /orders/submit`) |
| Gas per intent | User pays gas to lock | Gasless after initial deposit |
| Settlement | `finalise` (no signature needed) | `finaliseWithSignature` (signatures required) |
| Order type | N/A (detected on-chain) | `CatalystCompactOrder` |

***

## Flow

**1. Deposit into The Compact**

Deposit tokens into The Compact resource lock. This is a one-time on-chain transaction. Later intents draw from this balance and cost no extra gas.

See the [lintent.org demo](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/lifiintent/tx.ts#L199) for a UI example of depositing and registering intents.

**2. Request a quote**

Call the legacy V0 endpoint `POST /quote/request` with `"oif-resource-lock-v0"` in `supportedTypes` to indicate Compact support. The [V1 quote endpoint](https://docs.li.fi/lifi-intents/intents-api/request-quote.md) does not serve resource-lock flows; it only returns user-open orders. So this flow uses V0, which takes [EIP-7930 interoperable addresses](https://eips.ethereum.org/EIPS/eip-7930) — these pack the chain and address into one bytes field. Build them with the [`interoperable-addresses`](https://www.npmjs.com/package/interoperable-addresses) library.

```ts TypeScript theme={"system"}
const response = await fetch('https://order.li.fi/quote/request', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user: '0x0001000002210514YOUR_WALLET_ADDRESS_20_BYTES',
    intent: {
      intentType: 'oif-swap',
      inputs: [{
        user: '0x0001000002210514YOUR_WALLET_ADDRESS_20_BYTES',
        asset: '0x0001000002210514833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',  // USDC on Base
        amount: '10000000',
      }],
      outputs: [{
        receiver: '0x0001000002A4B114YOUR_WALLET_ADDRESS_20_BYTES',
        asset: '0x0001000002A4B114af88d065e77c8cC2239327C5EDb3A432268e5831',  // USDC on Arbitrum
        amount: null,
      }],
      swapType: 'exact-input',
    },
    supportedTypes: ['oif-escrow-v0', 'oif-resource-lock-v0'],
  }),
});

const { quotes } = await response.json();
const bestQuote = quotes[0];
```

**3. Sign the claim**

Compact orders are authorized via EIP-712 signatures rather than on-chain transactions. Sign a `BatchClaim` using The Compact's domain separator.

```ts TypeScript theme={"system"}
// The claim is signed as a BatchCompact struct:
// struct BatchCompact {
//   address arbiter;        // InputSettlerCompact address
//   address sponsor;        // User address
//   uint256 nonce;          // Allocator nonce
//   uint256 expires;        // Claim expiry
//   uint256[2][] idsAndAmounts;
//   Mandate mandate;        // fillDeadline, inputOracle, outputs
// }
```

For a complete signing example, see the [lintent.org demo](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/lifiintent/tx.ts#L144).

**4. Submit the order off-chain**

Submit the signed order to `POST /orders/submit`. Include the `quoteId` for preferential solver matching.

```bash curl theme={"system"}
curl -X POST 'https://order.li.fi/orders/submit' \
  -H 'Content-Type: application/json' \
  -d '{
    "orderType": "CatalystCompactOrder",
    "inputSettler": "0x0000000000cd5f7fDEc90a03a31F79E5Fbc6A9Cf",
    "quoteId": "QUOTE_ID_FROM_STEP_2",
    "order": {
      "expires": 1942819670,
      "user": "0xYOUR_WALLET_ADDRESS",
      "nonce": "1004",
      "originChainId": "8453",
      "fillDeadline": 1942819670,
      "inputOracle": "0x008C3800F3Ad9b3B662d002E90Cc00000000eE17",
      "inputs": [
        ["749071750893463290574776461331093852760741783827", "10000000"]
      ],
      "outputs": [{
        "oracle": "0x000000000000000000000000008C3800F3Ad9b3B662d002E90Cc00000000eE17",
        "settler": "0x00000000000000000000000075220B7600c300005038432a0000f308e0000068",
        "chainId": "42161",
        "token": "0x000000000000000000000000af88d065e77c8cC2239327C5EDb3A432268e5831",
        "amount": "9986765",
        "recipient": "0x000000000000000000000000YOUR_WALLET_ADDRESS",
        "callbackData": "0x",
        "context": "0x"
      }]
    }
  }'
```

> [!WARNING]
>
> `CatalystCompactOrder` submissions require a sponsor signature or Compact registration tx hash. The payload above is a reference template. You must include the appropriate signature fields. Always test on `order-dev.li.fi` before production.

The response includes a `meta` object with `orderIdentifier` and `onChainOrderId` for tracking.

**5. Track the order**

Same as the escrow flow. Poll `GET /orders/status` until the order reaches `Settled` or `Refunded`.

```ts TypeScript theme={"system"}
const trackOrder = async (orderId: string) => {
  let status: string;
  do {
    const res = await fetch(
      `https://order.li.fi/orders/status?catalystOrderId=${orderId}`
    );
    const data = await res.json();
    status = data.meta.orderStatus;
    console.log(`Status: ${status}`);

    if (status !== 'Settled' && status !== 'Refunded') {
      await new Promise(r => setTimeout(r, 3000));
    }
  } while (status !== 'Settled' && status !== 'Refunded');

  return status;
};

// `submitted`: the JSON response body from POST /orders/submit in the previous step
await trackOrder(submitted.meta.orderIdentifier);
```

***

## Integration Examples

* [Signing a BatchCompact claim](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/lifiintent/tx.ts#L144) (UI example)
* [Depositing and registering intents](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/lifiintent/tx.ts#L199) (UI example)
* [RegisterIntentLib.sol](https://github.com/catalystsystem/catalyst-intent/blob/27ce0972c150ed113f66ae91069eb953f23d920b/src/libs/RegisterIntentLib.sol#L100-L131) (smart contract example)

***

## Next Steps

> [card] [Settlement](https://docs.li.fi/lifi-intents/architecture/settlement.md#input-settlement)
>
> Compact claim structure, registration, and Permit2 details

> [card] [Resource Locks](https://docs.li.fi/lifi-intents/knowledge-database/resource-locks.md)
>
> How resource locks, sponsors, allocators, and arbiters work

> [card] [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md)
>
> Full order construction reference and validation rules

> [card] [Track Order Status](https://docs.li.fi/lifi-intents/intents-api/track-status.md)
>
> Monitor orders via the API or on-chain events


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.