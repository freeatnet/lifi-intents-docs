> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Quoting Orders

> Publish standing quotes, supported routes, prices, and amount ranges to the order server.

The order server delivers quotes to intent issuers, built from the inventory that solvers have published. Instead of answering each quote request, you push your inventory to the order server as routes and refresh it as markets move.

```ts theme={"system"}
// POST /quotes/submit

interface Body {
  quotes: Quote[]
}

interface Quote {
    expiry: number;               // unix seconds
    fromChain: string;            // network id, e.g. "10"
    fromAsset: string;            // chain-native address (hex on EVM, base58 on Solana)
    fromDecimals: number;
    toChain: string;
    toAsset: string;
    toDecimals: number;
    ranges: QuoteRange[];
    exclusiveFor?: string;        // solver address granted exclusivity
    integratorKeyHash?: string;   // sha256 of an integrator key; serves only that integrator
}

interface QuoteRange {
    minAmount: string;            // raw base units, inclusive
    maxAmount: string;            // raw base units, inclusive
    quote: string;                // price: toAsset per 1 fromAsset, unscaled
    fixedCost?: string;           // fee added in fromAsset units
}
```

When providing quotes, there are some important considerations to keep in mind:

* **expiry** is the user-side quote expiry, not the solver-side expiry. If a user requests a 30-second intent, only quotes expiring after 30 seconds are returned.
* **fromDecimals** and **toDecimals** are not validated by the order server; it only uses them to translate the quote between assets as a real exchange rate.
* **ranges** lists the price tiers you will fill at, one tier per band of input amounts. Your prices should include gas costs.
* **exclusiveFor** is optional. When the order names it, only that address can fill the order until the exclusive deadline. Without it, any solver can fill. Orders built from your quote count toward your [reputation](https://docs.li.fi/lifi-intents/for-solvers/reputation.md#orders-that-count) either way, so leaving it out only gives up the protected window.
* **quote** is written like a normal price — do not scale for decimals; the order server applies `fromDecimals`/`toDecimals` for you.
* **integratorKeyHash**, if set, makes the quote visible only to requests carrying the matching `X-Integrator-Key`; if omitted, the quote serves only keyless (open-market) requests.

The example below quotes USDC from Optimism to Arbitrum.

```json theme={"system"}
{
  "quotes": [{
    "expiry": 1757673110,
    "fromChain": "10",
    "toChain": "42161",
    "fromAsset": "0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85",
    "toAsset": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "fromDecimals": 6,
    "toDecimals": 6,
    "ranges": [
      {
        "minAmount": "10",
        "maxAmount": "100",
        "quote": "0.95"
      },
      {
        "minAmount": "100",
        "maxAmount": "10000",
        "quote": "0.995"
      },
      {
        "minAmount": "10000",
        "maxAmount": "50000",
        "quote": "0.99"
      }
    ]
  }]
}
```

> [!NOTE]
>
> **You can overwrite old quotes by sending new quotes**
>
> To avoid solvers sending short-lived quotes, the order server overwrites existing quotes whenever it receives a new one. This allows you to have long-lived quotes and only update them as markets evolve. If you send a quote with an expiry in the past, it will still overwrite old quotes and immediately expire.

By broadcasting quotes in this manner, solvers can:

* Publish the liquidity they have across many chain and token pairs.
* Set their own pricing and fee structure.
* Set the transaction sizes they will take.
* Update their quotes as market conditions change.

The order server uses broadcasted quotes to match user requests without querying solvers in real time, resulting in faster responses and less overhead.

Submit quotes to [`/quotes/submit`](https://order.li.fi/docs#tag/solver-api/post/quotes/submit).

### Solana Routes

Quotes for Solana routes use the same schema. Three details differ:

* `fromChain` / `toChain` take **bare network-id strings** (`"1151111081099710"` for Solana mainnet, `"1151111081099712"` for devnet) — not the CAIP-2 `solana:` prefixed ids used by the registration and supported-contracts endpoints.
* Assets are chain-native: base58 mint addresses on Solana, hex on EVM.
* `exclusiveFor` is the solver identity that can claim the inputs, so it uses the **input chain's** address format. For Solana → EVM, submit the solver's base58 Solana public key; the order encodes its raw 32 bytes into the EVM output context.

The following example quotes USDC from Solana mainnet to Base:

```json theme={"system"}
{
  "quotes": [{
    "expiry": 1800000000,
    "fromChain": "1151111081099710",
    "toChain": "8453",
    "fromAsset": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
    "toAsset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "fromDecimals": 6,
    "toDecimals": 6,
    "ranges": [
      { "minAmount": "1000000", "maxAmount": "1000000000", "quote": "0.999" }
    ]
  }]
}
```

## Supported Contracts

> [!NOTE]
>
> Supported Contracts is experimental. Standard quote requests do not require these declarations. If you are working with us on integrator contract allowlists, custom settlers, or oracles, we will tell you which entries to provide.

LI.FI Intent is a modular system based on four components: output settler, output oracle, input oracle, and input settler. How you interact with them can differ from one component to the next. As a result, the order server needs to know which components each solver supports — your quotes are only eligible for routes covered by your declared set.

```ts theme={"system"}
// PUT /api/v1/solver/supported-contracts   (GET to read the current set)

interface Body {
  oracle: ChainContract[];
  inputSettler: ChainContract[];
  outputSettler: ChainContract[];
}

interface ChainContract {
    chain: string;    // CAIP-2, e.g. "eip155:10" or "solana:1151111081099710"
    address: string;  // chain-native address
}
```

`PUT` replaces the full set, so include every contract you support on each update.

For Solana, declare the programs you support with CAIP-2 `solana:<id>` chains and base58 program IDs in the same three roles (`oracle`, `inputSettler`, `outputSettler`).

## Registering Accounts

Register every account you fill from — EVM and Solana — so the order server can link your on-chain activity to your solver identity for [reputation tracking](https://docs.li.fi/lifi-intents/for-solvers/reputation.md). Registration proves you own the account by signing a server-issued message:

**1. Fetch a one-time message**

`GET /api/v1/solver/register/message` returns `Register account for solver <solverId>. Nonce: <uuid>.` — valid for 24 hours, single-use.

**2. Sign it with the account's key**

EVM accounts sign it as a standard Ethereum personal message — EIP-191, what wallet `signMessage` produces — hex-encoded, or an EIP-1271 signature for smart-contract wallets. Solana accounts sign the raw message bytes with ed25519 and base58-encode the signature.

**3. Submit the registration**

`POST /api/v1/solver/register` with the account, CAIP-2 chain, message, and signature.

```bash cURL theme={"system"}
message=$(curl -s 'https://order.li.fi/api/v1/solver/register/message' \
  -H "x-api-key: $SOLVER_API_KEY" | jq -r '.data.message')

# sign $message with your account's key, then:
jq -n \
  --arg account "8YLKoCu7NwqHNS8GzuvA2ibsvLrsg22YMfMDafxh1B15" \
  --arg chain "solana:1151111081099710" \
  --arg message "$message" \
  --arg signature "<base58 ed25519 signature>" \
  '{account: $account, chain: $chain, message: $message, signature: $signature}' |
curl -X POST 'https://order.li.fi/api/v1/solver/register' \
  -H 'Content-Type: application/json' \
  -H "x-api-key: $SOLVER_API_KEY" \
  --data @-
```

```ts EVM theme={"system"}
// npm install viem
import { privateKeyToAccount } from "viem/accounts";

const account = privateKeyToAccount(process.env.SOLVER_PRIVATE_KEY! as `0x${string}`);

const headers = {
  "Content-Type": "application/json",
  "x-api-key": process.env.SOLVER_API_KEY!,
};

const { data } = await (
  await fetch("https://order.li.fi/api/v1/solver/register/message", { headers })
).json();

const signature = await account.signMessage({ message: data.message });

await fetch("https://order.li.fi/api/v1/solver/register", {
  method: "POST",
  headers,
  body: JSON.stringify({
    account: account.address,
    chain: "eip155:8453",
    message: data.message,
    signature,
  }),
});
```

```ts Solana theme={"system"}
// npm install tweetnacl bs58 @solana/web3.js
import { Keypair } from "@solana/web3.js";
import nacl from "tweetnacl";
import bs58 from "bs58";

const keypair = Keypair.fromSecretKey(bs58.decode(process.env.SOLVER_SECRET_KEY!));

const headers = {
  "Content-Type": "application/json",
  "x-api-key": process.env.SOLVER_API_KEY!,
};

const { data } = await (
  await fetch("https://order.li.fi/api/v1/solver/register/message", { headers })
).json();

const signature = bs58.encode(
  nacl.sign.detached(new TextEncoder().encode(data.message), keypair.secretKey),
);

await fetch("https://order.li.fi/api/v1/solver/register", {
  method: "POST",
  headers,
  body: JSON.stringify({
    account: keypair.publicKey.toBase58(),
    chain: "solana:1151111081099710",
    message: data.message,
    signature,
  }),
});
```

An account can only be registered by one solver (`403` if already taken); `POST /api/v1/solver/unregister` removes it. The message is consumed on the first `POST` attempt regardless of outcome — if registration fails, fetch a fresh message before retrying.

> [!NOTE]
>
> Keep secret keys and API keys in environment variables or a secret store — never in client-side code or version control.

## Authentication

Solver API endpoints require an API key passed via the `x-api-key` header. Your API key ties your quotes to your solver, so the order server can track whether you fill the orders you win. See [Reputation](https://docs.li.fi/lifi-intents/for-solvers/reputation.md).

See [Authentication](https://docs.li.fi/lifi-intents/authentication.md#solvers-api-key-required) for registration and key management details.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.