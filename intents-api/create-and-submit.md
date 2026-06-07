> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Create and Submit Orders

> Construct a StandardOrder, choose a resource lock, attach optional calldata, and submit to the LI.FI Intents solver network.

An order (intent) describes what the user wants: input assets, desired outputs, settlement parameters, and deadlines. Once constructed, submit it to the order server for distribution to the solver network.

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
  call: `0x${string}`;
  context: `0x${string}`;
};
```

### Field Reference

| Field           | Description                                                                                                                                                                 |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `user`          | Owner of the input assets. Refund recipient if the order is not filled.                                                                                                     |
| `nonce`         | Unique identifier per order. For The Compact, this is an allocator nonce.                                                                                                   |
| `originChainId` | Chain where the intent originates                                                                                                                                           |
| `expires`       | Deadline (unix timestamp) for submitting proof that the order was filled. After this, the order can be refunded.                                                            |
| `fillDeadline`  | Deadline for the solver to fill the order. Must be before `expires`.                                                                                                        |
| `inputOracle`   | Oracle address on the origin chain. Must correspond to `outputs[].oracle`.                                                                                                  |
| `inputs`        | Array of `[tokenIdentifier, amount]` pairs as `uint256`. For escrow, the token identifier is the address cast to `uint256`. For The Compact, it is the ERC-6909 identifier. |
| `outputs`       | Array of `MandateOutput` describing desired deliveries                                                                                                                      |

### Output Fields

| Field       | Description                                                                              |
| ----------- | ---------------------------------------------------------------------------------------- |
| `oracle`    | Oracle address on the output chain. Must be in the same oracle network as `inputOracle`. |
| `settler`   | Output settler contract. Determines auction type and settlement logic.                   |
| `chainId`   | Destination chain identifier                                                             |
| `token`     | Token identifier on the destination chain                                                |
| `amount`    | Output token amount                                                                      |
| `recipient` | Recipient address for the output tokens and any calldata                                 |
| `call`      | Calldata delivered to `recipient` after token delivery. Set to `0x` if not needed.       |
| `context`   | Auction parameters and settlement context. Set to `0x` for simple limit orders.          |

***

## Choosing a Resource Lock

Resource locks determine how user funds are held while the solver fills the order.

| Lock                                     | Best for                                                  | How it works                                                                                        |
| ---------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Simple Escrow** (`InputSettlerEscrow`) | Standard integrations alongside non-resource-lock bridges | User approves the escrow contract (via direct approval or Permit2). Tokens are escrowed per-intent. |
| **The Compact** (`InputSettlerCompact`)  | Resource-lock-native applications                         | Long-lived deposit. Users deposit once and can issue multiple intents from the same balance.        |

If you're adding LI.FI Intents alongside other bridges, Simple Escrow is recommended. If you're building a resource-lock-first application, consider The Compact. See [Input Settlement](/lifi-intents/architecture/input-settlement) for registration details.

***

## Submitting an Order

For non-gasless escrow integrations, submit orders on-chain (for example through `open` / `openFor` flow) and use the order server for quote discovery + status tracking.

Use `POST /orders/submit` only when your integration needs off-chain order submission (for example gasless escrow or signed Compact claims). If you used a quote from the order server, include the `quoteId` for preferential processing.

<CodeGroup>
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
        "inputOracle": "0x0000003E06000007A224AeE90052fA6bb46d43C9",
        "inputs": [
          ["749071750893463290574776461331093852760741783827", "10000000"]
        ],
        "outputs": [{
          "oracle": "0x0000000000000000000000000000003E06000007A224AeE90052fA6bb46d43C9",
          "settler": "0x0000000000000000000000000000000000eC36B683C2E6AC89e9A75989C22a2e",
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
</CodeGroup>

The response includes the full order, associated quote (if any), and a `meta` object with `orderIdentifier` and `onChainOrderId` for tracking.

<Warning>
  This is a reference template. `CatalystCompactOrder` submissions require a sponsor signature or Compact registration tx hash. Always validate on `order-dev.li.fi` before mainnet rollout.
</Warning>

***

## On-Chain vs Off-Chain Orders

| Type                    | Examples                                         | Order server required?                                       |
| ----------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| **On-chain**            | Escrow opened intents, registered Compact claims | No. Solvers and the order server detect these automatically. |
| **Off-chain (gasless)** | Gasless escrow intents, signed Compact claims    | Yes. Submit via `POST /orders/submit`.                       |

<Note>
  The order server is a convenience layer, not a requirement for on-chain orders. All on-chain intents can be submitted without it. Some solvers detect on-chain intents independently.
</Note>

***

## Calls on Delivery

Each `MandateOutput` supports a `call` field for calldata executed on the destination chain after token delivery. The recipient receives tokens first, then the output settler calls `outputFilled(bytes32 token, uint256 amount, bytes executionData)` on the `recipient` contract.

When using calls on delivery:

* Tokens are delivered before the call executes for each individual output
* If the call fails, the entire intent cannot be filled
* The recipient must be a contract that implements `outputFilled`, not an EOA
* For multi-chain outputs, only the first output should include calldata

For arbitrary contract calls, use a batching contract or Single-Call Architecture (SCA) wrapper since the call is wrapped in `outputFilled`. Contact LI.FI if you need an SCA implementation.

***

## Intent Validation

Before submitting, validate your order against these rules.

### Security

1. Ensure the oracle network is secure relative to the intent value
2. All oracles (input and output) should belong to the same network
3. Multi-output orders can be vulnerable to DoS by filling only the first output. Make the first output the most valuable.
4. Calls from the output settler cannot be trusted for authentication since anyone can fill outputs
5. Set `user` as the intended refund recipient, not the intent issuer or relay address.

<Warning>
  Ensure that the refund recipient can properly receive and handle funds. Using an address that cannot accept tokens (e.g. a contract without a receive function, or the wrong chain address) will result in permanent loss of funds.
</Warning>

### Correctness

1. Use known, reliable tokens, settlement contracts, and oracles
2. Use the fewest tokens possible. Low-value or obscure tokens reduce fill likelihood.
3. Provide sufficient time between `fillDeadline` and `expires` to accommodate messaging delays
4. Ensure the nonce is unique. Escrow requires user uniqueness, while The Compact requires allocator uniqueness.
5. For The Compact, ensure all inputs share the same allocator and the resource lock expiry is after the intent expiry

***

## Next Steps

<CardGroup cols={2}>
  <Card title="Track Order Status" icon="signal" href="/lifi-intents/intents-api/track-status">
    Monitor order progress and handle terminal states
  </Card>

  <Card title="Request a Quote" icon="bolt" href="/lifi-intents/intents-api/request-quote">
    Fetch solver pricing before constructing an order
  </Card>

  <Card title="Input Settlement" icon="lock" href="/lifi-intents/architecture/input-settlement">
    Escrow and Compact registration details
  </Card>

  <Card title="Oracle Systems" icon="satellite-dish" href="/lifi-intents/architecture/oracle-systems">
    Oracle networks and verification
  </Card>
</CardGroup>
