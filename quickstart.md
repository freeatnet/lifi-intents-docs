> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Quickstart

> Request a quote, open an escrow order on-chain, and track it to completion with the LI.FI Intents API.

This quickstart walks you through a complete cross-chain USDC transfer (Base → Arbitrum) using the **standard escrow flow**, the recommended path for most integrations. By the end, you'll have requested a quote, approved tokens, opened an order on-chain, and tracked it to settlement.

The quote endpoint returns a ready-to-sign transaction that opens the order — you don't construct anything yourself.

The entire flow uses TypeScript with [viem](https://viem.sh/) since the escrow flow requires on-chain transactions. You'll need a public client connected to Base and an account (wallet) with USDC. The same flow works from Tron and Solana origins — only the signing library changes; see [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md).

> [!NOTE]
>
> No API key is required. All integrator endpoints are open with generous rate limits. See [Authentication](https://docs.li.fi/lifi-intents/authentication.md) for details.

***

## Prerequisites

The escrow flow involves on-chain transactions (approving tokens and sending the open transaction). We use [viem](https://viem.sh/) to interact with the Base network, but any EVM-compatible library works the same way.

```bash theme={"system"}
npm install viem
```

```ts TypeScript theme={"system"}
import { createPublicClient, createWalletClient, http, type Hex } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';
import { base } from 'viem/chains';

const account = privateKeyToAccount('<YOUR_PRIVATE_KEY>' as Hex);
const publicClient = createPublicClient({ chain: base, transport: http('https://mainnet.base.org') });
const walletClient = createWalletClient({ account, chain: base, transport: http('https://mainnet.base.org') });
```

***

## Escrow Flow

The standard escrow integration is a 4-step process: request a quote, approve tokens, send the quote's open transaction, and track the order to settlement.

**1. Request a quote**

Call `POST /api/v1/integrator/quote/request` with the user's input and desired output as CAIP-2 chain + native address pairs. This example sends 10 USDC from Base to Arbitrum.

```ts TypeScript theme={"system"}
const userAddress = account.address;

const response = await fetch('https://order.li.fi/api/v1/integrator/quote/request', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    user: { chain: 'eip155:8453', address: userAddress },
    intent: {
      intentType: 'oif-swap',
      inputs: [{
        chain: 'eip155:8453',
        user: userAddress,
        asset: '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',  // USDC on Base
        amount: '10000000',  // 10 USDC (6 decimals)
      }],
      outputs: [{
        chain: 'eip155:42161',
        receiver: userAddress,
        asset: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831',  // USDC on Arbitrum
        amount: null,
      }],
      swapType: 'exact-input',
    },
    supportedTypes: ['oif-user-open-v0'],
  }),
});

const { quotes } = await response.json();
const bestQuote = quotes[0];
console.log('Output amount:', bestQuote.preview.outputs[0].amount);
```

The response contains an array of quotes sorted by best price; the best quote is at index 0. Each quote carries `order.openIntentTx` — the transaction that opens the order — and `order.checks.allowances`, the approvals it needs. See [Request a Quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md) for the full request and response reference.

**2. Approve tokens**

Cover every entry in `order.checks.allowances`: the input settler needs permission to transfer your tokens when the order opens.

```ts TypeScript theme={"system"}
import { erc20Abi } from 'viem';

for (const check of bestQuote.order.checks.allowances) {
  const currentAllowance = await publicClient.readContract({
    address: check.token,
    abi: erc20Abi,
    functionName: 'allowance',
    args: [userAddress, check.spender],
  });

  if (currentAllowance < BigInt(check.required)) {
    const approveHash = await walletClient.writeContract({
      address: check.token,
      abi: erc20Abi,
      functionName: 'approve',
      args: [check.spender, BigInt(check.required)],
    });
    await publicClient.waitForTransactionReceipt({ hash: approveHash });
    console.log('Approval confirmed');
  }
}
```

> [!NOTE]
>
> You can also use [Permit2](https://github.com/Uniswap/permit2) for gasless approvals. The escrow supports deposits via Permit2 signatures. See [Escrow Deposit Methods](https://docs.li.fi/lifi-intents/architecture/settlement.md#escrow-deposit-methods) for details.

**3. Send the open transaction**

Send the transaction from the quote. It calls `open()` on the input settler, which locks your tokens in escrow and broadcasts the intent to solvers.

```ts TypeScript theme={"system"}
const { openIntentTx } = bestQuote.order;

const openHash = await walletClient.sendTransaction({
  to: openIntentTx.to,
  data: openIntentTx.data,
});
const receipt = await publicClient.waitForTransactionReceipt({ hash: openHash });
console.log('Order opened! Tx:', receipt.transactionHash);

// keccak256 of the settler's event signature:
// Open(bytes32 indexed orderId, StandardOrder order)
const OPEN_EVENT_TOPIC =
  '0x9ff74bd56d00785b881ef9fa3f03d7b598686a39a9bcff89a6008db588b18a7b';

const openLog = receipt.logs.find((log) => log.topics[0] === OPEN_EVENT_TOPIC);
const orderId = openLog?.topics[1];
if (!orderId) throw new Error('Open event not found in receipt logs');
console.log('Order ID:', orderId);
```

The `open()` call transfers your tokens into escrow and emits an `Open` event. Solvers and the order server detect this event automatically. No separate submission step is needed.

> [!NOTE]
>
> Want to construct the `StandardOrder` yourself instead — custom deadlines, multiple outputs, calls on delivery? See [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md) for the order structure and validation rules.

**4. Track the order**

Poll `GET /orders/status` until the order reaches a terminal state. Use the `onChainOrderId` from the `Open` event.

```ts TypeScript theme={"system"}
const trackOrder = async (onChainOrderId: string) => {
  let status: string;
  do {
    const res = await fetch(
      `https://order.li.fi/orders/status?onChainOrderId=${onChainOrderId}`
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

await trackOrder(orderId);
```

| Status | Meaning |
| - | - |
| `Signed` | Order accepted and available for solver pickup |
| `Delivered` | Solver has delivered assets on the destination chain |
| `Settled` | Oracle verified delivery, locked funds released to solver. Terminal. |
| `Refunded` | Order was not filled and the input funds were returned to the user. Terminal. |

Once `Settled`, the user has received USDC on Arbitrum and the solver has been paid from escrow. `Refunded` is the alternate terminal state: no solver filled the order and the input funds went back to the user.

Your order reports `Signed` from the moment the order server sees it, including right after the `Open` transaction confirms — opening does not produce a distinct status. See [Track Order Status](https://docs.li.fi/lifi-intents/intents-api/track-status.md#status-lifecycle) for the full lifecycle.

***

## What Just Happened

1. **Requested a quote.** The order server returned pricing from its solver network, plus a ready-to-sign transaction that opens the order.
2. **Approved tokens.** You authorized the escrow contract to transfer your USDC, following the quote's allowance checks.
3. **Opened the order.** Sending the quote's transaction locked tokens in escrow and broadcast the intent to solvers via the `Open` event.
4. **Solver delivered.** A solver fulfilled the order by delivering USDC on Arbitrum.
5. **Settlement completed.** The oracle verified delivery and the escrow released the locked funds to the solver.

***

## Next Steps

> [card] [API Overview](https://docs.li.fi/lifi-intents/intents-api/api-overview.md)
>
> Full endpoint reference, base URLs, and authentication

> [card] [Request a Quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md)
>
> Exact-input, exact-output, and the Solana and Tron varieties

> [card] [Compact Orders](https://docs.li.fi/lifi-intents/intents-api/compact-orders.md)
>
> Off-chain gasless order submission via The Compact

> [card] [Track Order Status](https://docs.li.fi/lifi-intents/intents-api/track-status.md)
>
> On-chain events and order server status polling


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.