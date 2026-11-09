> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Intent / Solver Marketplace

> LI.FI's intent marketplace for cross-chain and same-chain transfers, where the order server matches user intents against solver standing quotes.

LI.FI Intent / Solver Marketplace helps wallets and applications execute same-chain and cross-chain token transfers through a solver network. Solvers publish standing quotes (price curves for routes and amount ranges), and the order server matches intents to available solver inventory.

> [!NOTE]
>
> LI.FI Intent/Solver Marketplace is the official foundation of the [Open Intents Framework (OIF)](https://openintents.xyz/), an open-source effort led by the Ethereum Foundation to make interoperability permissionless.

***

## What You Get as an Integrator

* **Competitive pricing.** Intents are matched against solver standing quotes and routed to the best available fill path.
* **Fast settlement.** Solvers use their own capital to deliver assets on the destination chain before settlement completes.
* **Same-chain and cross-chain support.** Route availability updates continuously based on active solver quoting.
* **Optional gasless order submission.** Off-chain order submission is gasless, while funding actions (approval/deposit into Escrow or Compact) are still on-chain transactions.
* **Calls on delivery.** Attach calldata to intents for custom logic after asset delivery (EVM, self-constructed orders).
* **Composer cross-chain flows.** From the Composer API, an Intents order ([`lifiIntents.order`](https://docs.li.fi/composer/composer-api/ops/lifi-intents-order)) can deliver into an escrow, and Composer runs more steps on the destination chain once the funds land. See [Cross-chain flows](https://docs.li.fi/composer/composer-api/concepts/cross-chain-flows.md).

***

## How It Works

1. **User expresses an intent.** The user deposits tokens into an input settler (escrow or Compact resource lock), then states the output they want, without saying how to get it.
2. **Intent is distributed.** The lock generates a proof of deposit, which is broadcast to the order server and solver network.
3. **Intent is matched.** The order server matches the intent against existing solver quotes (standing inventory), not live per-intent RFQ bidding.
4. **Solver delivers.** The winning solver delivers the requested assets on the destination chain.
5. **Verification and settlement.** The output settler records the fill and generates attestations; the oracle/input settler verifies delivery and releases locked funds.

![LI.FI Intent Lifecycle Diagram](https://mintcdn.com/lifi/pOTDBSGpduSCqu2-/images/lifi-intent-small.png?fit=max\&auto=format\&n=pOTDBSGpduSCqu2-\&q=85\&s=add276a85219988e9d26317552afb8ef)

> [!NOTE]
>
> The same lifecycle runs on non-EVM chains. On Solana, the settlement layer is a set of Anchor programs and orders are opened on-chain by signing the quote's returned transaction — see [Solana vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md) for what differs. Tron differences are covered in [Tron vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/tron-deltas.md).

***

## Supported Chains

Same-chain intents are live across all supported chains. Coverage includes the major pairs and a wide range of long-tail tokens, Ondo assets among them. For the most up-to-date chain support, call [`GET /chains/supported`](https://order.li.fi/docs#tag/bridge-api/GET/chains/supported). Testnet chains are also available on the same endpoint.

A supported chain means the protocol is deployed there; whether a specific pair can be executed depends on active solver quoting. Check [supported routes](#supported-routes) before assuming a route is available.

## Supported Routes

Route availability updates continuously based on active solver quoting. For the full route list including token-level detail, call [`GET /api/v1/integrator/routes`](https://order.li.fi/docs#tag/bridge-api-v1/GET/api/v1/integrator/routes) — it returns CAIP-2 chain ids and token metadata, optionally filtered by source and destination chain.

***

## Authentication

Integrator endpoints are fully open with no API key and generous rate limits. Solvers must register for an API key via the solver UI. See [Authentication](https://docs.li.fi/lifi-intents/authentication.md) for details.

***

## Key Concepts

| Concept | Description |
| - | - |
| **Intent** | A user request that expresses the desired outcome without specifying execution details |
| **Solver** | An entity that publishes standing quotes and fulfills intents using its own capital and liquidity |
| **Input Settler** | Umbrella term for source-side settlement contracts that release locked funds after verified delivery |
| **Escrow (InputSettlerEscrow)** | Per-intent lock model where user funds are escrowed per order |
| **Compact (InputSettlerCompact)** | Deposit-based model where users can fund once and issue multiple intents from allocated balances |
| **Order Server** | Off-chain matching layer that maps intents to available solver quotes and distributes order flow |
| **Output Settler** | Destination-side contract that accepts solver fills and generates attestations used for verification + settlement |
| **Oracle** | The layer that verifies delivery on the destination chain so settlement can happen on the source chain. For same-chain intents, the output settler itself (OutputSettlerSimple) acts as the oracle, so no cross-chain proof has to travel. |
| **StandardOrder** | Canonical on-chain order struct used for intents |
| **`fillDeadline` vs `expires`** | `fillDeadline` is solver fill cutoff; `expires` is final deadline after which refunds can be processed |
| **Order types** | Supported order types include limit, exclusive limit, Dutch auction, and exclusive Dutch auction |

For full definitions, see the [Glossary](https://docs.li.fi/lifi-intents/knowledge-database/glossary.md).

***

## Next Steps

> [card] [Quickstart](https://docs.li.fi/lifi-intents/quickstart.md)
>
> Request a quote, submit an order, and track it in under 5 minutes

> [card] [Intents API Overview](https://docs.li.fi/lifi-intents/intents-api/api-overview.md)
>
> Endpoints, base URLs, authentication, and order lifecycle

> [card] [Solver API](https://docs.li.fi/lifi-intents/for-solvers/api-overview.md)
>
> Solver endpoints, quoting, and filling orders

> [card] [System Architecture](https://docs.li.fi/lifi-intents/architecture/overview.md)
>
> Settlers, oracles, and contract addresses


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.