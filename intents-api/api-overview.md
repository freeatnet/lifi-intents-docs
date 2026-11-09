> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# API Overview

> Base URLs, endpoints, authentication, and order lifecycle for the LI.FI Intents API.

The Intents API is the integrator-facing interface for requesting quotes, submitting orders, and tracking status. All endpoints are open and require no API key.

> [!NOTE]
>
> For solver-facing endpoints (quote submission, account management), see the [Solver API Overview](https://docs.li.fi/lifi-intents/for-solvers/api-overview.md). For authentication details, see [Authentication](https://docs.li.fi/lifi-intents/authentication.md#integrators).

***

## Base URLs

| Environment | URL | Use |
| - | - | - |
| Production | `https://order.li.fi` | Live orders with real assets |
| Development | `https://order-dev.li.fi` | Testnet orders (Sepolia, Base Sepolia, etc.) |

Interactive API documentation is available at [`/docs`](https://order.li.fi/docs) on both environments.

***

## Endpoints

| Method | Path | Description |
| - | - | - |
| `POST` | `/api/v1/integrator/quote/request` | Request a quote for an intent (CAIP-2 + native addresses, all chains). Returns a ready-to-sign origin-chain transaction. |
| `GET` | `/api/v1/integrator/routes` | Discover active routes, optionally filtered by source and destination chain (CAIP-2) |
| `GET` | `/orders` | List orders with optional filters |
| `GET` | `/orders/status` | Get order status by `onChainOrderId` or `catalystOrderId` |
| `GET` | `/chains/supported` | Get the list of supported chains |
| `POST` | `/orders/submit` | Submit an off-chain order — gasless escrow and signed Compact claims, EVM only |

### Platform Differences

The quote and tracking journey is the same on every chain: request a V1 quote, sign and send the returned transaction, poll `/orders/status`. The differences are at the edges:

* **EVM**: satisfy `checks.allowances` before sending; off-chain flows (gasless escrow, signed Compact claims) can instead go through `POST /orders/submit`. See [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md).
* **Tron**: wrap the returned calldata in a `TriggerSmartContract` envelope; escrow settlement only. See [Tron vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/tron-deltas.md).
* **Solana**: rebuild the transaction template with the returned compute-unit limit and a fresh blockhash before signing; on-chain only — `POST /orders/submit` does not accept Solana orders. See [Solana vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md).

***

## Order Lifecycle

`GET /orders/status` reports one of four states:

```
Signed → Delivered → Settled
```

| Status | Meaning |
| - | - |
| `Signed` | Order accepted and available for solver pickup |
| `Delivered` | Solver has delivered assets on the destination chain |
| `Settled` | Oracle verified delivery, locked funds released to solver. Terminal. |
| `Refunded` | Order was not filled and the input funds were returned to the user. Terminal. |

`Signed` covers everything before a fill, including the on-chain `Open` event. If the order is not filled before `fillDeadline`, it can be refunded after `expires`, ending in `Refunded`. See [Track Order Status](https://docs.li.fi/lifi-intents/intents-api/track-status.md#status-lifecycle) for polling and the WebSocket status mapping.

***

## Addresses

All endpoints take addresses as CAIP-2 chain id + native address pairs (`{ "chain": "eip155:8453", "address": "0x…" }`) — hex on EVM, base58 on Tron and Solana. No special encoding is required.

***

## Amounts

All amounts are in the token's smallest unit (wei-equivalent), with no decimals. For example, 1.5 USDC (6 decimals) is `1500000`.

***

## Next Steps

> [card] [Request a Quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md)
>
> Fetch solver pricing for your intent

> [card] [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md)
>
> Construct and submit an order to the solver network

> [card] [Track Order Status](https://docs.li.fi/lifi-intents/intents-api/track-status.md)
>
> Monitor orders via the API or on-chain events

> [card] [Solver API](https://docs.li.fi/lifi-intents/for-solvers/api-overview.md)
>
> Solver endpoints, quoting, and filling orders


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.