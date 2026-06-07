> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Authentication

> API keys, access levels, and environments for the LI.FI Intents API.

The LI.FI Intents API has two access levels: open integrator endpoints and authenticated solver endpoints.

***

## Integrators

Integrator endpoints are fully open. No API key, no registration, no rate limits. You can start building immediately.

### Endpoints

| Method | Path                               | Description                              |
| ------ | ---------------------------------- | ---------------------------------------- |
| `POST` | `/quote/request`                   | Request a quote for an intent            |
| `GET`  | `/api/v1/integrator/quote/request` | V1 integrator-friendly quote endpoint    |
| `POST` | `/orders/submit`                   | Submit an order to the solver network    |
| `GET`  | `/orders`                          | List orders with optional filters        |
| `GET`  | `/orders/status`                   | Get order status                         |
| `GET`  | `/chains/supported`                | Get the list of supported chains         |
| `GET`  | `/routes`                          | Get supported routes (chain/asset pairs) |

***

## Solvers (API Key Required)

Solver endpoints require an API key passed via the `api-key` header. These endpoints are used for submitting quotes, managing solver accounts, and tracking reputation.

| Method | Path                             | Description                                   |
| ------ | -------------------------------- | --------------------------------------------- |
| `POST` | `/quotes/submit`                 | Submit or update up to 200K solver quotes     |
| `GET`  | `/solver-api/quotes`             | Get solver's uploaded quote inventory         |
| `GET`  | `/solver-api/solver/identities`  | Get solver identities for reputation tracking |
| `POST` | `/solver-api/account/register`   | Register a solver account                     |
| `POST` | `/solver-api/account/unregister` | Unregister a solver account                   |

### Authenticating Requests

Pass your API key in the `api-key` header:

```bash theme={"system"}
curl -X POST 'https://order.li.fi/quotes/submit' \
  -H 'Content-Type: application/json' \
  -H 'api-key: YOUR_SOLVER_API_KEY' \
  -d '{ ... }'
```

### Getting an API Key

Register and manage your solver API key in the solver UI:

| Environment | Solver UI                                     | API Base URL              |
| ----------- | --------------------------------------------- | ------------------------- |
| Production  | [intents.li.fi](https://intents.li.fi/)       | `https://order.li.fi`     |
| Testnet     | [devintents.li.fi](https://devintents.li.fi/) | `https://order-dev.li.fi` |

We recommend starting on testnet to validate your integration before going live.

***

## Environments

| Environment | Base URL                  | Use                                          |
| ----------- | ------------------------- | -------------------------------------------- |
| Production  | `https://order.li.fi`     | Live orders with real assets                 |
| Development | `https://order-dev.li.fi` | Testnet orders (Sepolia, Base Sepolia, etc.) |

Interactive API documentation is available at [`/docs`](https://order.li.fi/docs) on both environments.

***

## Next Steps

<CardGroup cols={2}>
  <Card title="Quickstart" icon="rocket" href="/lifi-intents/quickstart">
    End-to-end escrow flow for integrators
  </Card>

  <Card title="Intents API Overview" icon="code" href="/lifi-intents/intents-api/api-overview">
    Integrator endpoints, order lifecycle, and address encoding
  </Card>

  <Card title="Solver API Overview" icon="gears" href="/lifi-intents/for-solvers/api-overview">
    Solver endpoints, quoting, and account management
  </Card>
</CardGroup>
