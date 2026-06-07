> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Solver API Overview

> Endpoints, authentication, and integration surface for LI.FI Intents solvers.

The Solver API is the authenticated interface for solvers to submit quotes, manage accounts, and track reputation. All solver endpoints require an API key.

<Note>
  For integrator-facing endpoints (quotes, orders, status tracking), see the [Intents API Overview](/lifi-intents/intents-api/api-overview).
</Note>

***

## Base URLs

| Environment | Base URL                  | Solver UI                                     |
| ----------- | ------------------------- | --------------------------------------------- |
| Production  | `https://order.li.fi`     | [intents.li.fi](https://intents.li.fi/)       |
| Testnet     | `https://order-dev.li.fi` | [devintents.li.fi](https://devintents.li.fi/) |

Interactive API documentation is available at [`/docs`](https://order.li.fi/docs) on both environments.

***

## Authentication

All Solver API endpoints require an API key passed via the `api-key` header. Register for a key in the solver UI linked above. See [Authentication](/lifi-intents/authentication#solvers-api-key-required) for full details.

```bash theme={"system"}
curl -X POST 'https://order.li.fi/quotes/submit' \
  -H 'Content-Type: application/json' \
  -H 'api-key: YOUR_SOLVER_API_KEY' \
  -d '{ ... }'
```

***

## Endpoints

### Quote Management

| Method | Path                 | Description                                       |
| ------ | -------------------- | ------------------------------------------------- |
| `POST` | `/quotes/submit`     | Submit or update up to 200K solver quotes at once |
| `GET`  | `/solver-api/quotes` | Retrieve your currently uploaded quote inventory  |

Quotes define the routes, price curves, and amount ranges you're willing to fill. The order server uses your quotes to match intents, so you don't need to respond to individual quote requests. See [Quoting Orders](/lifi-intents/for-solvers/quoting) for the full quote format and strategy.

### Account Management

| Method | Path                             | Description                 |
| ------ | -------------------------------- | --------------------------- |
| `POST` | `/solver-api/account/register`   | Register a solver account   |
| `POST` | `/solver-api/account/unregister` | Unregister a solver account |

### Reputation

| Method | Path                            | Description                                              |
| ------ | ------------------------------- | -------------------------------------------------------- |
| `GET`  | `/solver-api/solver/identities` | Get solver identities registered for reputation tracking |

See [Reputation](/lifi-intents/for-solvers/reputation) for how solver performance is tracked.

### Order Collection

Solvers receive orders via two channels:

1. **WebSocket (recommended):** Connect to the order server for real-time order streaming.
2. **On-chain monitoring:** Watch for `Open` events on the `InputSettlerEscrow` or `InputSettlerCompact` contracts.

See [Order Flow](/lifi-intents/for-solvers/orderflow) for connection details and order validation.

***

## Solver Flow Summary

```
Register account → Submit quotes → Receive orders → Fill outputs → Validate via oracle → Settle
```

| Step         | What you do                                                | Reference                                                   |
| ------------ | ---------------------------------------------------------- | ----------------------------------------------------------- |
| **Register** | Create a solver account via `/solver-api/account/register` | This page                                                   |
| **Quote**    | Push inventory to `/quotes/submit`                         | [Quoting](/lifi-intents/for-solvers/quoting)                |
| **Receive**  | Connect to WebSocket or monitor on-chain events            | [Order Flow](/lifi-intents/for-solvers/orderflow)           |
| **Fill**     | Deliver assets via `OutputSettlerSimple.fill()`            | [Filling Orders](/lifi-intents/for-solvers/filling-orders)  |
| **Prove**    | Submit fill proof through the oracle system                | [Oracle Systems](/lifi-intents/architecture/oracle-systems) |
| **Settle**   | Call `finalise` on the input settler to receive payment    | [Settlement](/lifi-intents/for-solvers/settlement)          |

***

## Next Steps

<CardGroup cols={2}>
  <Card title="Solving LI.FI Intents" icon="gears" href="/lifi-intents/for-solvers/intro">
    Full solver integration guide with order types and filling
  </Card>

  <Card title="Quoting Orders" icon="chart-line" href="/lifi-intents/for-solvers/quoting">
    Quote format, price curves, and inventory management
  </Card>

  <Card title="Order Flow" icon="arrows-split-up-and-left" href="/lifi-intents/for-solvers/orderflow">
    WebSocket connection, order validation, and on-chain monitoring
  </Card>

  <Card title="Authentication" icon="key" href="/lifi-intents/authentication#solvers-api-key-required">
    API key registration and environment details
  </Card>
</CardGroup>
