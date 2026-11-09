> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Authentication

> API keys, access levels, and environments for the LI.FI Intents API.

The LI.FI Intents API has two access levels: open integrator endpoints and authenticated solver endpoints.

***

## Integrators

Integrator endpoints are fully open. No API key, no registration, and generous rate limits. You can start building immediately.

### Endpoints

| Method | Path | Description |
| - | - | - |
| `POST` | `/api/v1/integrator/quote/request` | Request a quote for an intent (all chains) |
| `GET` | `/api/v1/integrator/routes` | Discover active routes (CAIP-2) |
| `GET` | `/orders` | List orders with optional filters |
| `GET` | `/orders/status` | Get order status |
| `GET` | `/chains/supported` | Get the list of supported chains |
| `POST` | `/orders/submit` | Submit an off-chain order (gasless / Compact, EVM only) |

***

## Solvers (API Key Required)

Solver endpoints require an API key passed via the `x-api-key` header. Use these endpoints to submit quotes, manage solver accounts, and track reputation.

| Method | Path | Description |
| - | - | - |
| `POST` | `/quotes/submit` | Submit or update up to 200K solver quotes |
| `GET` | `/solver-api/quotes` | Get solver's uploaded quote inventory |
| `GET` | `/solver-api/solver/identities` | Get solver identities for reputation tracking |
| `GET` | `/api/v1/solver/register/message` | Get a one-time registration message to sign |
| `POST` | `/api/v1/solver/register` | Register a solver account (EVM ECDSA, EIP-1271, Solana ed25519) |
| `POST` | `/api/v1/solver/unregister` | Unregister a solver account |
| `GET`/`PUT` | `/api/v1/solver/supported-contracts` | Read or replace the list of oracle and settler contracts your solver supports |
| `POST` | `/solver-api/account/register` | Legacy V0 account registration (EVM signatures only) |
| `POST` | `/solver-api/account/unregister` | Legacy V0 account removal |
| `GET` | `/api/v1/solver/reliability` | Get your current status, active pauses and trials, and miss counts |
| `GET` | `/api/v1/solver/reliability/pauses` | List your pause history |

### Authenticating Requests

Pass your API key in the `x-api-key` header:

```bash theme={"system"}
curl -X POST 'https://order.li.fi/quotes/submit' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_SOLVER_API_KEY' \
  -d '{ ... }'
```

### Getting an API Key

Register and manage your solver API key in the solver UI:

| Environment | Solver UI | API Base URL |
| - | - | - |
| Production | [intents.li.fi](https://intents.li.fi/) | `https://order.li.fi` |
| Testnet | [devintents.li.fi](https://devintents.li.fi/) | `https://order-dev.li.fi` |

We recommend starting on testnet to validate your integration before going live.

***

## Environments

| Environment | Base URL | Use |
| - | - | - |
| Production | `https://order.li.fi` | Live orders with real assets |
| Development | `https://order-dev.li.fi` | Testnet orders (Sepolia, Base Sepolia, etc.) |

Interactive API documentation is available at [`/docs`](https://order.li.fi/docs) on both environments.

***

## Next Steps

> [card] [Quickstart](https://docs.li.fi/lifi-intents/quickstart.md)
>
> End-to-end escrow flow for integrators

> [card] [Intents API Overview](https://docs.li.fi/lifi-intents/intents-api/api-overview.md)
>
> Integrator endpoints, order lifecycle, and address encoding

> [card] [Solver API Overview](https://docs.li.fi/lifi-intents/for-solvers/api-overview.md)
>
> Solver endpoints, quoting, and account management


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.