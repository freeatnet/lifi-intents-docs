> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Track Order Status

> Monitor LI.FI intent orders via the order server API or on-chain events.

After submitting an order, you can track its progress through the order server API or by monitoring on-chain events directly.

***

## Order Server Status

Call `GET /orders/status` with either the `onChainOrderId` or the `catalystOrderId` returned when you submitted the order.

<CodeGroup>
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
</CodeGroup>

### Query Parameters

| Parameter         | Description                                                                      |
| ----------------- | -------------------------------------------------------------------------------- |
| `onChainOrderId`  | The on-chain order ID emitted in contract events                                 |
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
    orderStatus: "Submitted" | "Open" | "Signed" | "Delivered" | "Settled",
    destinationAddress: string,
    orderIdentifier: string,
    onChainOrderId: string,
    signedAt: string | null,
    deliveredAt: string | null,
    settledAt: string | null,
    expiredAt: string | null,
    orderInitiatedTxHash: string | null,
    orderDeliveredTxHash: string | null,
    orderVerifiedTxHash: string | null,
    orderSettledTxHash: string | null,
    solverAddress: string | null
  }
}
```

### Status Lifecycle

| Status      | Meaning                                                          |
| ----------- | ---------------------------------------------------------------- |
| `Submitted` | Order received by the order server (transitional)                |
| `Open`      | Order registered on-chain (transitional)                         |
| `Signed`    | Order signed and available for solver pickup                     |
| `Delivered` | Solver has delivered assets on the destination chain             |
| `Settled`   | Proof verified, locked funds released to solver. Order complete. |

Most integrations should treat `Signed`, `Delivered`, and `Settled` as the primary execution states. `Submitted` and `Open` may not be visible in every path.

### Polling Example

```ts theme={"system"}
const pollUntilDone = async (orderId: string) => {
  let data;
  do {
    data = await getOrderStatus(orderId);
    const { orderStatus } = data.meta;
    console.log(`Status: ${orderStatus}`);

    if (orderStatus === 'Settled') {
      console.log('Complete. Delivery tx:', data.meta.orderDeliveredTxHash);
      return data;
    }

    if (data.meta.expiredAt) {
      console.log('Order expired.');
      return data;
    }

    await new Promise(r => setTimeout(r, 3000));
  } while (true);
};
```

***

## Listing Orders

Use `GET /orders` to query multiple orders with filters.

```bash theme={"system"}
curl -X GET 'https://order.li.fi/orders?user=0xYOUR_ADDRESS&status=Delivered&limit=10'
```

| Parameter         | Description                         |
| ----------------- | ----------------------------------- |
| `user`            | Filter by initiator address         |
| `status`          | Filter by order status              |
| `nonce`           | Filter by nonce                     |
| `exclusiveFor`    | Filter by solver address            |
| `onChainOrderId`  | Filter by on-chain order ID         |
| `catalystOrderId` | Filter by order server ID           |
| `limit`           | Max results (1-50, default 50)      |
| `offset`          | Results to skip (0-1000, default 0) |

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

***

## Next Steps

<CardGroup cols={2}>
  <Card title="API Overview" icon="code" href="/lifi-intents/intents-api/api-overview">
    Full endpoint reference and base URLs
  </Card>

  <Card title="Create and Submit Orders" icon="paper-plane" href="/lifi-intents/intents-api/create-and-submit">
    Order construction and submission
  </Card>

  <Card title="Settlement" icon="lock" href="/lifi-intents/architecture/settlement">
    How input and output settlement works
  </Card>

  <Card title="System Overview" icon="diagram-project" href="/lifi-intents/architecture/overview">
    Full architecture of the Intent/Solver Marketplace
  </Card>
</CardGroup>
