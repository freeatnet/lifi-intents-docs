> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# API Overview

> Base URLs, endpoints, authentication, and order lifecycle for the LI.FI Intents API.

The Intents API is the integrator-facing interface for requesting quotes, submitting orders, and tracking status. All endpoints are open and require no API key.

<Note>
  For solver-facing endpoints (quote submission, account management), see the [Solver API Overview](/lifi-intents/for-solvers/api-overview). For authentication details, see [Authentication](/lifi-intents/authentication#integrators).
</Note>

***

## Base URLs

| Environment | URL                       | Use                                          |
| ----------- | ------------------------- | -------------------------------------------- |
| Production  | `https://order.li.fi`     | Live orders with real assets                 |
| Development | `https://order-dev.li.fi` | Testnet orders (Sepolia, Base Sepolia, etc.) |

Interactive API documentation is available at [`/docs`](https://order.li.fi/docs) on both environments.

***

## Endpoints

| Method | Path                               | Description                                                 |
| ------ | ---------------------------------- | ----------------------------------------------------------- |
| `POST` | `/quote/request`                   | Request a quote for an intent (exact-input or exact-output) |
| `GET`  | `/api/v1/integrator/quote/request` | V1 quote endpoint for integrator-friendly quote parameters  |
| `POST` | `/orders/submit`                   | Submit an order (intent) to the solver network              |
| `GET`  | `/orders`                          | List orders with optional filters                           |
| `GET`  | `/orders/status`                   | Get order status by `onChainOrderId` or `catalystOrderId`   |
| `GET`  | `/chains/supported`                | Get the list of supported chains                            |
| `GET`  | `/routes`                          | Get supported routes (chain/asset pairs)                    |

***

## Order Lifecycle

Orders can include transitional states depending on submission path and indexing timing. In most integrations, the primary execution progression is:

```
Signed → Delivered → Settled
```

| Status        | What happened                                             |
| ------------- | --------------------------------------------------------- |
| **Signed**    | Order signed and broadcast to the solver network          |
| **Delivered** | Solver has delivered assets on the destination chain      |
| **Settled**   | Oracle verified delivery, locked funds released to solver |

If the order is not filled before `fillDeadline`, it can be refunded after `expires`.

***

## Interoperable Addresses

The Intents API uses [EIP-7930 interoperable addresses](https://eips.ethereum.org/EIPS/eip-7930), which encode both the chain and address in a single bytes field. This applies to `user`, `asset`, and `receiver` fields in quote requests.

Example encoding of `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` on Base (chain ID 8453):

```
0x0001|0000|02|2105|14|833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
  ^^^^  Version: 1
       ^^^^  ChainType: EVM
            ^^  Chain Reference length: 2 bytes
               ^^^^  Chain Reference: 8453 (hex 2105)
                    ^^  Address length: 20 bytes
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  Address
```

***

## Amounts

All amounts are in the token's smallest unit (wei-equivalent), with no decimals. For example, 1.5 USDC (6 decimals) is `"1500000"`.

***

## Next Steps

<CardGroup cols={2}>
  <Card title="Request a Quote" icon="bolt" href="/lifi-intents/intents-api/request-quote">
    Fetch solver pricing for your intent
  </Card>

  <Card title="Create and Submit Orders" icon="paper-plane" href="/lifi-intents/intents-api/create-and-submit">
    Construct and submit an order to the solver network
  </Card>

  <Card title="Track Order Status" icon="signal" href="/lifi-intents/intents-api/track-status">
    Monitor orders via the API or on-chain events
  </Card>

  <Card title="Solver API" icon="gears" href="/lifi-intents/for-solvers/api-overview">
    Solver endpoints, quoting, and filling orders
  </Card>
</CardGroup>
