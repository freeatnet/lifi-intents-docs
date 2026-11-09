> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Create and Submit Orders

> Construct a StandardOrder, choose a resource lock, and get it to the solver network — by opening it on-chain or submitting it to the order server.

An order (intent) describes what the user wants: input assets, desired outputs, settlement parameters, and deadlines. On every chain, the recommended path is the same: request a [V1 quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md), then sign and send the ready-made open transaction it returns. Off-chain submission via the order server exists for EVM gasless flows. Calls on delivery are available where the destination settler supports them — on EVM, not on Solana.

***

## Order Structure

LI.FI intents use the OIF `StandardOrder`, which supports single-chain inputs and multi-chain outputs.

```ts theme={"system"}
type StandardOrder = {
  user: `0x${string}`;
  nonce: bigint;
  originChainId: bigint;
  expires: number;
  fillDeadline: number;
  inputOracle: `0x${string}`;
  inputs: [bigint, bigint][];
  outputs: MandateOutput[];
};

type MandateOutput = {
  oracle: `0x${string}`;
  settler: `0x${string}`;
  chainId: bigint;
  token: `0x${string}`;
  amount: bigint;
  recipient: `0x${string}`;
  call: `0x${string}`; // named `callbackData` in the order server's REST payloads
  context: `0x${string}`;
};
```

### Field Reference

| Field | Description |
| - | - |
| `user` | Owner of the input assets. Refund recipient if the order is not filled. |
| `nonce` | Unique identifier per order. For The Compact, this is an allocator nonce. |
| `originChainId` | Chain where the intent originates |
| `expires` | Deadline (unix timestamp) for submitting proof that the order was filled. After this, the order can be refunded. |
| `fillDeadline` | Deadline for the solver to fill the order. Must be before `expires`. |
| `inputOracle` | Oracle address on the origin chain. Must correspond to `outputs[].oracle`. |
| `inputs` | Array of `[tokenIdentifier, amount]` pairs as `uint256`. For escrow, the token identifier is the address cast to `uint256`. For The Compact, it is the ERC-6909 identifier. |
| `outputs` | Array of `MandateOutput` describing desired deliveries |

### Output Fields

| Field | Description |
| - | - |
| `oracle` | Oracle address on the output chain. Must be in the same oracle network as `inputOracle`. |
| `settler` | Output settler contract. Determines auction type and settlement logic. |
| `chainId` | Destination chain identifier |
| `token` | Token identifier on the destination chain |
| `amount` | Output token amount |
| `recipient` | Recipient address for the output tokens and any calldata |
| `call` | Calldata delivered to `recipient` after token delivery. Set to `0x` if not needed. |
| `context` | Auction parameters and settlement context. Set to `0x` for simple limit orders. |

***

## Choosing a Resource Lock

Resource locks determine how user funds are held while the solver fills the order.

| Lock | Best for | How it works |
| - | - | - |
| **Simple Escrow** (`InputSettlerEscrow`) | Standard integrations alongside non-resource-lock bridges | User approves the escrow contract (via direct approval or Permit2). Tokens are escrowed per-intent. |
| **The Compact** (`InputSettlerCompact`) | Resource-lock-native applications | Long-lived deposit. Users deposit once and can issue multiple intents from the same balance. |

If you're adding LI.FI Intents alongside other bridges, Simple Escrow is recommended. If you're building a resource-lock-first application, consider The Compact. See [Escrow Deposit Methods](https://docs.li.fi/lifi-intents/architecture/settlement.md#escrow-deposit-methods) for the ways funds can be deposited.

> [!NOTE]
>
> The deployed `InputSettlerCompact` contract has not yet been updated to the current version of the protocol. Its address will be published once the updated deployment is live — use Simple Escrow in the meantime, or contact LI.FI if you are building a Compact integration.

On Tron and Solana, only escrow is available — there is no Compact or resource-lock flow on either chain.

***

## Submitting an Order

Each [V1 quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md) returns `order.openIntentTx` — an origin-chain transaction that opens the order. Opening on-chain *is* the broadcast: the input settler emits an open event, the order server ingests it, and solvers pick the order up. There is no separate submission call.

The transaction's shape and the steps before sending differ per chain.

### Opening on EVM

`openIntentTx` carries the input settler address (`to`), hex calldata (`data`), and a gas estimate (`gasRequired`). Before sending, satisfy every entry in `order.checks.allowances` — each item asserts the user holds and has approved at least `required` of `token` for `spender` (direct ERC-20 approval or Permit2):

```ts theme={"system"}
import { erc20Abi } from 'viem';

// walletClient: your connected viem wallet client
// quote: the winning entry from the quote response
const { openIntentTx, checks } = quote.order;

for (const check of checks.allowances) {
  await walletClient.writeContract({
    address: check.token,
    abi: erc20Abi,
    functionName: 'approve',
    args: [check.spender, BigInt(check.required)],
  });
}

const hash = await walletClient.sendTransaction({
  to: openIntentTx.to,
  data: openIntentTx.data,
});
```

### Opening on Tron

On Tron, `openIntentTx.data` is full ABI calldata (selector + arguments), not a ready-to-broadcast transaction. Approve the TRC-20 input first, then wrap the calldata in a `TriggerSmartContract` envelope with TronWeb; `feeLimit` is a cap on energy spend in SUN, not an estimate:

```ts theme={"system"}
import { TronWeb } from 'tronweb';

const tronWeb = new TronWeb({ fullHost: 'https://api.trongrid.io' /*, privateKey */ });
const { to, data, feeLimit } = quote.order.openIntentTx;

const tx = await tronWeb.transactionBuilder.triggerSmartContract(
  to,                       // settler (base58)
  '',                       // selector empty: full calldata is supplied via `input`
  { feeLimit, input: data.replace(/^0x/, '') },
  [],
  tronWeb.defaultAddress.base58,
);
const signed = await tronWeb.trx.sign(tx.transaction);
await tronWeb.trx.sendRawTransaction(signed);
```

Details on address conversion and the TVM envelope are in [Tron vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/tron-deltas.md#on-chain-submission-tvm).

### Opening on Solana

On Solana, `openIntentTx.data` is a serialized transaction template. Before signing, replace its dummy blockhash and rebuild its message with the compute-unit limit returned in `openIntentTx.computeUnitsRequired`:

```ts theme={"system"}
import {
  ComputeBudgetProgram,
  Connection,
  Keypair,
  TransactionMessage,
  VersionedTransaction,
} from "@solana/web3.js";
import bs58 from "bs58";

const connection = new Connection("https://api.mainnet-beta.solana.com");
const userKeypair = Keypair.fromSecretKey(bs58.decode(process.env.USER_SECRET_KEY!));

// The winning entry from the quote response:
const quote = quotes[0];

const template = VersionedTransaction.deserialize(
  bs58.decode(quote.order.openIntentTx.data),
);
const message = TransactionMessage.decompile(template.message);
message.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
message.instructions.unshift(
  ComputeBudgetProgram.setComputeUnitLimit({
    units: Number(quote.order.openIntentTx.computeUnitsRequired),
  }),
);

const tx = new VersionedTransaction(message.compileToV0Message());
tx.sign([userKeypair]);
const signature = await connection.sendTransaction(tx);
```

The current template has no address lookup tables, so it can be decompiled without supplying lookup-table accounts. If that changes, resolve every lookup table and pass it to `TransactionMessage.decompile` before rebuilding. In a dApp, use your wallet adapter's `signTransaction` instead of a `Keypair`. The `open` instruction escrows the input tokens and emits an `OpenEvent`; the order server ingests that event and distributes the order to solvers. There is no approval pre-step: `checks.allowances` is always `[]` because the SPL transfer happens inside the `open` instruction.

> [!NOTE]
>
> Store secrets like `USER_SECRET_KEY` in environment variables — never hardcode them or expose them in browser code.

### Off-Chain Submission (Gasless and Compact)

Use `POST /orders/submit` only when your integration needs off-chain order submission — gasless escrow or signed Compact claims, both EVM-only. It does not accept Tron or Solana orders. If you used a quote from the order server, include the `quoteId` for preferential processing.

```bash curl theme={"system"}
curl -X POST 'https://order.li.fi/orders/submit' \
  -H 'Content-Type: application/json' \
  -d '{
    "orderType": "CatalystCompactOrder",
    "inputSettler": "0x0000000000cd5f7fDEc90a03a31F79E5Fbc6A9Cf",
    "quoteId": "QUOTE_ID_FROM_STEP_1",
    "order": {
      "expires": 1942819670,
      "user": "0xYOUR_WALLET_ADDRESS",
      "nonce": "1004",
      "originChainId": "8453",
      "fillDeadline": 1942819670,
      "inputOracle": "0x008C3800F3Ad9b3B662d002E90Cc00000000eE17",
      "inputs": [
        ["749071750893463290574776461331093852760741783827", "10000000"]
      ],
      "outputs": [{
        "oracle": "0x000000000000000000000000008C3800F3Ad9b3B662d002E90Cc00000000eE17",
        "settler": "0x00000000000000000000000075220B7600c300005038432a0000f308e0000068",
        "chainId": "42161",
        "token": "0x000000000000000000000000af88d065e77c8cC2239327C5EDb3A432268e5831",
        "amount": "9986765",
        "recipient": "0x000000000000000000000000YOUR_WALLET_ADDRESS",
        "callbackData": "0x",
        "context": "0x"
      }]
    }
  }'
```

The response includes the full order, associated quote (if any), and a `meta` object with `orderIdentifier` and `onChainOrderId` for tracking.

> [!WARNING]
>
> This is a reference template. `CatalystCompactOrder` submissions require a sponsor signature or Compact registration tx hash. Always validate on `order-dev.li.fi` before mainnet rollout.

***

## On-Chain vs Off-Chain Orders

| Type | Examples | Order server required? |
| - | - | - |
| **On-chain** | Escrow opened intents, registered Compact claims | No. Solvers and the order server detect these automatically. |
| **Off-chain (gasless)** | Gasless escrow intents, signed Compact claims | Yes. Submit via `POST /orders/submit`. |

> [!NOTE]
>
> The order server is a convenience layer, not a requirement for on-chain orders. All on-chain intents can be submitted without it. Some solvers detect on-chain intents independently.

***

## Calls on Delivery

Each `MandateOutput` supports a `call` field for calldata executed on the destination chain after token delivery. The recipient receives tokens first, then the output settler calls `outputFilled(bytes32 token, uint256 amount, bytes executionData)` on the `recipient` contract.

When using calls on delivery:

* For each output, the output settler delivers the tokens before it runs the call
* If the call fails, the entire intent cannot be filled
* The recipient must be a contract that implements `outputFilled`, not an EOA
* For multi-chain outputs, only the first output should include calldata

For arbitrary contract calls, use a batching contract or Single-Call Architecture (SCA) wrapper since the call is wrapped in `outputFilled`. Contact LI.FI if you need an SCA implementation.

> [!NOTE]
>
> Calls on delivery are not supported on Solana. Orders with Solana outputs must leave the callback field empty; the output settler rejects fills otherwise.

***

## Intent Validation

Before submitting, validate your order against these rules.

### Security

1. Check that the oracle network is secure enough for the value of the intent
2. All oracles (input and output) should belong to the same network
3. Multi-output orders can be vulnerable to DoS by filling only the first output. Make the first output the most valuable.
4. Calls from the output settler cannot be trusted for authentication since anyone can fill outputs
5. Set `user` as the intended refund recipient, not the intent issuer or relay address.

> [!WARNING]
>
> Ensure that the refund recipient can properly receive and handle funds. Using an address that cannot accept tokens (e.g. a contract without a receive function, or the wrong chain address) will lose the funds permanently.

### Correctness

1. Use known, reliable tokens, settlement contracts, and oracles
2. Use the fewest tokens possible. Low-value or obscure tokens make a fill less likely.
3. Leave enough time between `fillDeadline` and `expires` for messaging delays
4. Ensure the nonce is unique. Escrow requires user uniqueness, while The Compact requires allocator uniqueness.
5. For The Compact, ensure all inputs share the same allocator and the resource lock expiry is after the intent expiry

***

## Next Steps

> [card] [Track Order Status](https://docs.li.fi/lifi-intents/intents-api/track-status.md)
>
> Monitor order progress and handle terminal states

> [card] [Request a Quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md)
>
> Fetch solver pricing before constructing an order

> [card] [Settlement](https://docs.li.fi/lifi-intents/architecture/settlement.md#input-settlement)
>
> Escrow and Compact registration details

> [card] [Oracle Systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md)
>
> Oracle networks and verification


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.