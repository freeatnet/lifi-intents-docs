> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Solver API Overview

> Endpoints, authentication, and integration surface for LI.FI Intents solvers.

The Solver API is the authenticated interface for solvers to submit quotes, manage accounts, and track reputation. All solver endpoints require an API key.

> [!NOTE]
>
> For integrator-facing endpoints (quotes, orders, status tracking), see the [Intents API Overview](https://docs.li.fi/lifi-intents/intents-api/api-overview.md).

***

## Base URLs

| Environment | Base URL | Solver UI |
| - | - | - |
| Production | `https://order.li.fi` | [intents.li.fi](https://intents.li.fi/) |
| Testnet | `https://order-dev.li.fi` | [devintents.li.fi](https://devintents.li.fi/) |

Interactive API documentation is available at [`/docs`](https://order.li.fi/docs) on both environments.

***

## Authentication

All Solver API endpoints require an API key passed via the `x-api-key` header. Register for a key in the solver UI linked above. See [Authentication](https://docs.li.fi/lifi-intents/authentication.md#solvers-api-key-required) for full details.

```bash theme={"system"}
curl -X POST 'https://order.li.fi/quotes/submit' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_SOLVER_API_KEY' \
  -d '{ ... }'
```

> [!NOTE]
>
> Keep your API key out of client-side code and version control — load it from an environment variable or secret store.

***

## Endpoints

### Quote Management

| Method | Path | Description |
| - | - | - |
| `POST` | `/quotes/submit` | Submit or update up to 200K solver quotes at once |
| `GET` | `/solver-api/quotes` | Retrieve your currently uploaded quote inventory |

Quotes define the routes, price curves, and amount ranges you're willing to fill. The order server uses your quotes to match intents, so you don't need to respond to individual quote requests. See [Quoting Orders](https://docs.li.fi/lifi-intents/for-solvers/quoting.md) for the full quote format and strategy.

### Account Management

Register the accounts you fill from so the order server can link your on-chain activity to your solver identity:

| Method | Path | Description |
| - | - | - |
| `GET` | `/api/v1/solver/register/message` | Get a one-time message to sign (valid 24 h, single-use) |
| `POST` | `/api/v1/solver/register` | Register a solver account — EVM ECDSA, EIP-1271, or Solana ed25519 signatures |
| `POST` | `/api/v1/solver/unregister` | Unregister a solver account |

See [Registering Accounts](https://docs.li.fi/lifi-intents/for-solvers/quoting.md#registering-accounts) for the full walkthrough. The legacy V0 endpoints `/solver-api/account/register` and `/solver-api/account/unregister` (EVM signatures, no server-issued message) remain available for existing integrations.

### Supported Contracts

| Method | Path | Description |
| - | - | - |
| `GET` | `/api/v1/solver/supported-contracts` | Read your declared oracle / input-settler / output-settler set |
| `PUT` | `/api/v1/solver/supported-contracts` | Replace the full set (CAIP-2 `{chain, address}` entries) |

Your declared contract set decides which routes your quotes can serve.

### Reputation

| Method | Path | Description |
| - | - | - |
| `GET` | `/solver-api/solver/identities` | Get solver identities registered for reputation tracking |
| `GET` | [`/api/v1/solver/reliability`](https://order.li.fi/docs#tag/solver-api-v1/GET/api/v1/solver/reliability) | Get your current status, active pauses and trials, and miss counts |
| `GET` | [`/api/v1/solver/reliability/pauses`](https://order.li.fi/docs#tag/solver-api-v1/GET/api/v1/solver/reliability/pauses) | List your pause history, newest first |

See [Reputation](https://docs.li.fi/lifi-intents/for-solvers/reputation.md) for which orders count and when you can be paused.

### Order Collection

Solvers receive orders via two channels:

1. **WebSocket (recommended):** Connect to the order server for real-time order streaming.
2. **On-chain monitoring:** Watch for `Open` events on the `InputSettlerEscrow` or `InputSettlerCompact` contracts.

See [Order Flow](https://docs.li.fi/lifi-intents/for-solvers/orderflow.md) for connection details and order validation.

***

## Solver Flow Summary

```
Register account → Submit quotes → Receive orders → Fill outputs → Validate via oracle → Settle
```

| Step | What you do | Reference |
| - | - | - |
| **Register** | Register your accounts via `/api/v1/solver/register` | This page |
| **Quote** | Push inventory to `/quotes/submit` | [Quoting](https://docs.li.fi/lifi-intents/for-solvers/quoting.md) |
| **Receive** | Connect to WebSocket or monitor on-chain events | [Order Flow](https://docs.li.fi/lifi-intents/for-solvers/orderflow.md) |
| **Fill** | Deliver assets via `OutputSettlerSimple.fill()` | [Solving LI.FI Intents](https://docs.li.fi/lifi-intents/for-solvers/intro.md#filling-intents) |
| **Prove** | Submit fill proof through the oracle system | [Oracle Systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md) |
| **Settle** | Call `finalise` on the input settler to receive payment | [Settlement](https://docs.li.fi/lifi-intents/for-solvers/settlement.md) |

***

## Next Steps

> [card] [Solving LI.FI Intents](https://docs.li.fi/lifi-intents/for-solvers/intro.md)
>
> Full solver integration guide with order types and filling

> [card] [Quoting Orders](https://docs.li.fi/lifi-intents/for-solvers/quoting.md)
>
> Quote format, price curves, and inventory management

> [card] [Order Flow](https://docs.li.fi/lifi-intents/for-solvers/orderflow.md)
>
> WebSocket connection, order validation, and on-chain monitoring

> [card] [Authentication](https://docs.li.fi/lifi-intents/authentication.md#solvers-api-key-required)
>
> API key registration and environment details


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.