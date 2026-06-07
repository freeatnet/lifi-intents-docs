> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Quoting Orders

> LI.FI Intents enables efficient quoting and inventory management. Broadcast liquidity and pricing, compete on your terms, and respond to market changes instantly.

The order server delivers quotes to intent issuers. These are based on the available inventory provided by solvers. Instead of responding to each quote request, solvers can regularly push their inventory to the order server as routes.

```ts theme={"system"}
// POST /quotes/submit

interface Body {
  quotes: Quote[]
}

interface Quote {
    expiry: number;
    fromChainId: string;
    fromAsset: string;
    fromDecimals: number;
    toChainId: string;
    toAsset: string;
    toDecimals: number;
    ranges: QuoteRange[];
    maxToAmount?: bigint;
    exclusiveFor: string;
}

interface QuoteRange {
    minAmount: string;
    maxAmount: string;
    quote: string;
}
```

When providing quotes, there are some important considerations to keep in mind:

* **expiry** is the user-side quote expiry, not the solver-side expiry. If a user requests a 30-second intent, only quotes expiring after 30 seconds are returned.
* **fromDecimals** and **toDecimals** are not validated and are only used to translate the quote between assets as a real exchange rate.
* **maxToAmount** only needs to be provided if the solver does not want to quote their entire inventory at once. Within a single quote, this is the maximum allocation issued. If not provided, max(quoteRange.maxAmount) will be used instead.
* **ranges** is the list of descriptions of how the solver can fill assets. Quotes should include gas costs.
  The following code chunk provides an example quote of USDC from Optimism to Arbitrum.

```json theme={"system"}
{
  "quotes": [{
    "expiry": 1757673110,
    "fromChainId": "10",
    "toChainId": "42161",
    "fromAsset": "0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85",
    "toAsset": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "fromDecimals": "6",
    "toDecimals": "6",
    "ranges": [
      {
        "minAmount": "10",
        "maxAmount": "100",
        "quote": "0.95",
      },
      {
        "minAmount": "100",
        "maxAmount": "10000",
        "quote": "0.995",
      },
      {
        "minAmount": "10000",
        "maxAmount": "50000",
        "quote": "0.99",
      }
    ]
  }]
}
```

<Note>
  **You can overwrite old quotes by sending new quotes**

  To avoid solvers sending short-lived quotes, the order server will overwrite existing quotes when a new one is received. This allows you to have long-lived quotes and only update them as markets evolve. If you send a quote with an expiry in the past, it will still overwrite old quotes and immediately expire.
</Note>

By broadcasting quotes in this manner, solvers can:

* Efficiently communicate their available liquidity across multiple chain and token pairs.
* Set their own pricing and fee structure.
* Define limits on transaction sizes they are willing to process.
* Update their quotes as market conditions change.

The order server uses broadcasted quotes to match user requests without querying solvers in real time, resulting in faster responses and less overhead.

Submit quotes to [`/quotes/submit`](https://order.li.fi/docs#tag/solver-api/post/quotes/submit).

## Chain & Asset Support

The order server uses quotes to determine supported chains and assets. This allows solvers to add support for new chains and assets simply by producing quotes for them.

## Trust Components

LI.FI Intent is a modular system based on four components: output settler, output oracle, input oracle, and input settler. There may be differences in how these components are interacted with. As a result, it is important for the order server to know which components solvers support.

```ts theme={"system"}
// POST /quotes/trustComponents

interface Body {
  trustedComponents: Component[]
}

interface Component {
    address: string;
    chainId: string;
    forInput: bool;
    forOutput: bool;
    role: "settler" | "oracle"
}
```

Some address may be usable as both input and output component, thus it has to be provided explicitly where the compatibility is. Some common observations:

* Some oracles have different ways to fetch proofs depending on the origin chain. If an oracle has `forInput` as true, it means the solver knows how to deliver proofs. If an oracle has `forOutput` as true, it means the solver knows how to generate proofs. For oracles with no `inputOracle`(e.g., Polymer), use their`outputOracle` as descriptor.
* Input settlers and output settlers are not the same contract type. In most cases, either only `forInput` or only `forOutput` will be set.

More roles may be added in the future.

## Authentication

Solver API endpoints require an API key passed via the `api-key` header. Authentication links your API requests to your on-chain execution, allowing the order server to assign a reputation score.

See [Authentication](/lifi-intents/authentication#solvers-api-key-required) for registration and key management details.
