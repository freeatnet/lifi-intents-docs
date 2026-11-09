> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Track Order Status

> Monitor LI.FI intent orders via the order server API or on-chain events.

After submitting an order, you can track its progress through the order server API or by monitoring on-chain events directly.

***

## Order Server Status

Call `GET /orders/status` with either the `onChainOrderId` or the `catalystOrderId` returned when you submitted the order.

```bash curl theme={"system"}
curl -X GET 'https://order.li.fi/orders/status?catalystOrderId=intent_qVo7_1TkJ7VekL99O7SGS9cRHv0si6'
```

```ts TypeScript theme={"system"}
const getOrderStatus = async (orderId: string) => {
  const res = await fetch(
    `https://order.li.fi/orders/status?catalystOrderId=${orderId}`
  );
  return res.json();
};

const status = await getOrderStatus('intent_qVo7_1TkJ7VekL99O7SGS9cRHv0si6');
console.log('Status:', status.meta.orderStatus);
```

### Query Parameters

| Parameter | Description |
| - | - |
| `onChainOrderId` | The on-chain order ID emitted in contract events. For Solana, this is the `order_id` from the `OpenEvent`, hex-encoded. |
| `catalystOrderId` | The order server ID returned from `POST /orders/submit` (e.g. `intent_qVo7_...`) |

At least one parameter is required.

### Response

The response includes the full order, any associated quote, and a `meta` object with status information.

```ts theme={"system"}
{
  order: StandardOrder,
  quote: QuoteInfo | null,
  sponsorSignature: string | null,
  allocatorSignature: string | null,
  inputSettler: string,
  meta: {
    submitTime: number,
    orderStatus: "Signed" | "Delivered" | "Settled" | "Refunded",
    destinationAddress: string,
    orderIdentifier: string,
    onChainOrderId: string,
    signedAt: string | null,
    deliveredAt: string | null,
    settledAt: string | null,
    refundedAt: string | null,
    expiredAt: string | null,
    orderInitiatedTxHash: string | null,
    orderDeliveredTxHash: string | null,
    orderVerifiedTxHash: string | null,
    orderSettledTxHash: string | null,
    refundTxHash: string | null,
    lastCompactDepositBlockNumber: string | null,
    quoteId: string | null,
    solverAddress: string | null,
    integratorKeyHash?: string
  }
}
```

### Status Lifecycle

`meta.orderStatus` takes one of four values:

| Status | Meaning |
| - | - |
| `Signed` | Order accepted and available for solver pickup |
| `Delivered` | Solver has delivered assets on the destination chain |
| `Settled` | Oracle verified delivery, locked funds released to solver. Terminal. |
| `Refunded` | Order was not filled and the input funds were returned to the user. Terminal. |

The path is `Signed → Delivered → Settled`, with `Refunded` as the alternate terminal state.

`Signed` covers everything before a fill, including the on-chain `Open` event — opening an order does not produce a distinct status. Watch `meta.orderInitiatedTxHash` if you need to confirm the order landed on-chain.

> [!NOTE]
>
> `GET /orders` also accepts `Submitted` and `Open` as `?status=` filter values, but neither is ever returned in `meta.orderStatus` — orders matching those filters report as `Signed`.

REST responses and the solver-facing [WebSocket feed](https://docs.li.fi/lifi-intents/for-solvers/orderflow.md#order-server-events) use different names for the same statuses. The mapping:

| REST `meta.orderStatus` | WebSocket status |
| - | - |
| `Signed` | — (order announced as `user:vm-order-submit`) |
| `Delivered` | `Delivered` |
| `Settled` | `Finalised` |
| `Refunded` | `Refunded` |

In particular, do not wait for a WebSocket `Settled` event — the terminal success event is named `Finalised`.

### Polling Example

```ts theme={"system"}
const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));

const pollUntilDone = async (
  orderId: string,
  maxWaitAfterExpiryMs = 60 * 60 * 1000, // give up 1h after expiry
) => {
  let expirySeenAt: number | undefined;

  while (true) {
    const data = await getOrderStatus(orderId);
    const { orderStatus, expiredAt } = data.meta;
    // `order.expires` is a unix timestamp in seconds; don't rely on `expiredAt` alone.
    const expired = expiredAt !== null || Date.now() / 1000 > Number(data.order.expires);
    console.log(`Status: ${orderStatus}`);

    if (orderStatus === 'Settled') {
      console.log('Complete. Delivery tx:', data.meta.orderDeliveredTxHash);
      return { outcome: 'settled', data };
    }

    if (orderStatus === 'Refunded') {
      console.log('Refunded. Refund tx:', data.meta.refundTxHash);
      return { outcome: 'refunded', data };
    }

    if (expired) {
      // Expired means the order can no longer be filled, not that funds are back.
      expirySeenAt ??= Date.now();
      if (Date.now() - expirySeenAt > maxWaitAfterExpiryMs) {
        console.log('Expired and refund still pending.');
        return { outcome: 'refund-pending', data };
      }
      console.log('Expired, waiting for refund...');
      await sleep(30_000);
    } else {
      await sleep(3_000);
    }
  }
};
```

Once `order.expires` has passed (or `meta.expiredAt` is set), the order can no longer be filled — but that is not the same as a refund. The user's funds are back only when `orderStatus` is `Refunded` and `refundTxHash` is set, so keep polling after expiry (at a lower cadence) and surface a pending-refund state instead of reporting completion.

***

## Listing Orders

Use `GET /orders` to query multiple orders with filters. The order server caches responses for a few seconds, and it may throttle or temporarily time out aggressive polling.

```bash theme={"system"}
curl -X GET 'https://order.li.fi/orders?user=0xYOUR_ADDRESS&status=Delivered&limit=10'
```

| Parameter | Description |
| - | - |
| `user` | Filter by initiator address |
| `status` | Filter by order status |
| `nonce` | Filter by nonce |
| `exclusiveFor` | Filter by solver address |
| `onChainOrderId` | Filter by on-chain order ID |
| `catalystOrderId` | Filter by order server ID |
| `limit` | Max results (1-50, default 50) |
| `offset` | Results to skip (0-1000, default 0) |

***

## On-Chain Events

You can also track orders directly from contract events. The `orderId` is emitted as `topic1` on all relevant events.

### Order Opened

```solidity theme={"system"}
event Open(bytes32 indexed orderId, StandardOrder order);
```

### Output Filled (by solver)

```solidity theme={"system"}
event OutputFilled(
  bytes32 indexed orderId,
  bytes32 solver,
  uint32 timestamp,
  MandateOutput output,
  uint256 finalAmount
);
```

### Order Finalized (settlement complete)

```solidity theme={"system"}
event Finalised(
  bytes32 indexed orderId,
  bytes32 solver,
  bytes32 destination
);
```

### Order Refunded

```solidity theme={"system"}
event Refunded(bytes32 indexed orderId);
```

The on-chain `orderId` is deterministic and computed as:

```solidity theme={"system"}
function orderIdentifier(StandardOrder calldata order) internal view returns (bytes32) {
  return keccak256(
    abi.encodePacked(
      block.chainid,
      address(this),
      order.user,
      order.nonce,
      order.expires,
      order.fillDeadline,
      order.inputOracle,
      keccak256(abi.encodePacked(order.inputs)),
      abi.encode(order.outputs)
    )
  );
}
```

### Solana Events

On Solana, the programs emit Anchor events instead of Solidity events, one at every lifecycle step:

| Step | Event | Program | Fields |
| - | - | - | - |
| Order opened | `OpenEvent` | input settler | `order_id: [u8; 32]`, `order: StandardOrder` |
| Output filled | `OutputFilledEvent` | output settler | `settler`, `order_id`, `solver`, `timestamp`, `output: MandateOutput`, `final_amount` |
| Output not filled (after deadline) | `OutputNotFilledEvent` | output settler | `settler`, `order_id`, `fill_deadline`, `output: MandateOutput` |
| Cross-chain proven | `OutputProvenEvent` | oracle | `chain_id`, `remote_identifier`, `application`, `payload_hash` |
| Order finalised | `FinalisedEvent` | input settler | `settler`, `order_id`, `solver`, `destination` |
| Order refunded (expiry or proven non-fill) | `RefundedEvent` | input settler | `order_id` |

The Solana `order_id` is content-derived: `keccak256(borsh_serialize(StandardOrder))`. The `intents_protocol` program additionally emits attestation- and configuration-level events, listed in the [Attestations section of Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#events).

***

## Next Steps

> [card] [API Overview](https://docs.li.fi/lifi-intents/intents-api/api-overview.md)
>
> Full endpoint reference and base URLs

> [card] [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md)
>
> Order construction and submission

> [card] [Settlement](https://docs.li.fi/lifi-intents/architecture/settlement.md)
>
> How input and output settlement works

> [card] [System Overview](https://docs.li.fi/lifi-intents/architecture/overview.md)
>
> Full architecture of the Intent/Solver Marketplace


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.