> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Quickstart

> Request a quote, open an escrow order on-chain, and track it to completion with the LI.FI Intents API.

This quickstart walks you through a complete cross-chain USDC transfer (Base → Arbitrum) using the **standard escrow flow**, the recommended path for most integrations. By the end, you'll have requested a quote, approved tokens, opened an order on-chain, and tracked it to settlement.

If you need **gasless off-chain order submission** or are building on top of resource locks, see the [Compact Orders](/lifi-intents/intents-api/compact-orders) guide instead.

The entire flow uses TypeScript with [ethers.js](https://docs.ethers.org/v6/) since the escrow flow requires on-chain transactions. You'll need a provider connected to Base and a signer (wallet) with USDC.

<Note>
  No API key is required. All integrator endpoints are open with no rate limits. See [Authentication](/lifi-intents/authentication) for details.
</Note>

***

## Prerequisites

The escrow flow involves on-chain transactions (approving tokens and calling the escrow contract to lock funds). We use [ethers.js](https://docs.ethers.org/v6/) to interact with the Base network, but any EVM-compatible library (viem, web3.js, etc.) works the same way.

```bash theme={"system"}
npm install ethers
```

```ts TypeScript theme={"system"}
import { ethers } from 'ethers';

const provider = new ethers.JsonRpcProvider('https://mainnet.base.org');
const signer = new ethers.Wallet('YOUR_PRIVATE_KEY', provider);
```

The code below uses contract addresses from the [System Architecture](/lifi-intents/architecture/overview#smart-contracts) reference. These are the same across all supported chains.

***

## Escrow Flow

The standard escrow integration is a 4-step process: request a quote, approve tokens, open the order on-chain, and track it to settlement.

<Note>
  The Intents API uses [interoperable addresses (EIP-7930)](https://eips.ethereum.org/EIPS/eip-7930). See [Request a Quote](/lifi-intents/intents-api/request-quote) for encoding details.
</Note>

<Steps>
  <Step title="Request a quote">
    Call `POST /quote/request` with the user's input and desired output. This example sends 10 USDC from Base to Arbitrum.

    ```ts TypeScript theme={"system"}
    const userAddress = await signer.getAddress();
    const userAddressRaw = userAddress.slice(2); // Remove 0x prefix

    const response = await fetch('https://order.li.fi/quote/request', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user: `0x0001000002210514${userAddressRaw}`,
        intent: {
          intentType: 'oif-swap',
          inputs: [{
            user: `0x0001000002210514${userAddressRaw}`,
            asset: '0x0001000002210514833589fCD6eDb6E08f4c7C32D4f71b54bdA02913',  // USDC on Base
            amount: '10000000',  // 10 USDC (6 decimals)
          }],
          outputs: [{
            receiver: `0x0001000002A4B114${userAddressRaw}`,
            asset: '0x0001000002A4B114af88d065e77c8cC2239327C5EDb3A432268e5831',  // USDC on Arbitrum
            amount: null,
          }],
          swapType: 'exact-input',
        },
        supportedTypes: ['oif-escrow-v0'],
      }),
    });

    const { quotes } = await response.json();
    const bestQuote = quotes[0];
    console.log('Output amount:', bestQuote.preview.outputs[0].amount);
    ```

    The interoperable address prefix encodes the chain. `0x00010000022105` is Base (8453) and `0x0001000002A4B1` is Arbitrum (42161). See [Interoperable Address Encoding](/lifi-intents/intents-api/request-quote#interoperable-address-encoding) for details.

    The response contains an array of quotes sorted by best price. The best quote is at index 0. Use `preview.outputs[0].amount` from the best quote when constructing your order in the next steps.
  </Step>

  <Step title="Approve tokens">
    Before opening the escrow, the `InputSettlerEscrow` contract needs permission to transfer your tokens. Approve USDC on Base.

    ```ts TypeScript theme={"system"}
    const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
    const INPUT_SETTLER_ESCROW = '0x000025c3226C00B2Cdc200005a1600509f4e00C0';

    const erc20 = new ethers.Contract(USDC_BASE, [
      'function approve(address spender, uint256 amount) returns (bool)',
      'function allowance(address owner, address spender) view returns (uint256)',
    ], signer);

    const currentAllowance = await erc20.allowance(userAddress, INPUT_SETTLER_ESCROW);
    if (currentAllowance < BigInt('10000000')) {
      const tx = await erc20.approve(INPUT_SETTLER_ESCROW, '10000000');
      await tx.wait();
      console.log('Approval confirmed');
    }
    ```

    <Note>
      You can also use [Permit2](https://github.com/Uniswap/permit2) for gasless approvals. The escrow supports registration via Permit2 signatures. See [Input Settlement](/lifi-intents/architecture/input-settlement) for details.
    </Note>
  </Step>

  <Step title="Construct and open the order on-chain">
    Build a `StandardOrder` from the quote response and call `open()` on the `InputSettlerEscrow` contract. This locks your tokens and broadcasts the intent to solvers.

    ```ts TypeScript theme={"system"}
    const POLYMER_ORACLE = '0x0000003E06000007A224AeE90052fA6bb46d43C9';
    const OUTPUT_SETTLER = '0x0000000000eC36B683C2E6AC89e9A75989C22a2e';
    const USDC_ARBITRUM = '0xaf88d065e77c8cC2239327C5EDb3A432268e5831';

    const outputAmount = bestQuote.preview.outputs[0].amount;

    const order = {
      user: userAddress,
      nonce: BigInt(Date.now()),
      originChainId: BigInt(8453),
      expires: Math.floor(Date.now() / 1000) + 3600,        // 1 hour
      fillDeadline: Math.floor(Date.now() / 1000) + 1800,   // 30 minutes
      inputOracle: POLYMER_ORACLE,
      inputs: [
        [BigInt(USDC_BASE), BigInt('10000000')]              // [tokenId (address as uint256), amount]
      ],
      outputs: [{
        oracle: ethers.zeroPadValue(POLYMER_ORACLE, 32),
        settler: ethers.zeroPadValue(OUTPUT_SETTLER, 32),
        chainId: BigInt(42161),                              // Arbitrum
        token: ethers.zeroPadValue(USDC_ARBITRUM, 32),
        amount: BigInt(outputAmount),
        recipient: ethers.zeroPadValue(userAddress, 32),
        call: '0x',                                          // No callback
        context: '0x',                                       // Limit order (no auction)
      }],
    };

    const escrow = new ethers.Contract(INPUT_SETTLER_ESCROW, [
      'function open(bytes calldata order) external',
    ], signer);

    const encodedOrder = ethers.AbiCoder.defaultAbiCoder().encode(
      [
        'tuple(address user, uint256 nonce, uint256 originChainId, uint32 expires, uint32 fillDeadline, address inputOracle, uint256[2][] inputs, tuple(bytes32 oracle, bytes32 settler, uint256 chainId, bytes32 token, uint256 amount, bytes32 recipient, bytes call, bytes context)[] outputs)',
      ],
      [order]
    );

    const tx = await escrow.open(encodedOrder);
    const receipt = await tx.wait();
    console.log('Order opened! Tx:', receipt.hash);

    const orderId = receipt.logs[0]?.topics[1];
    console.log('Order ID:', orderId);
    ```

    The `open()` call transfers your tokens into escrow and emits an `Open` event. Solvers and the order server detect this event automatically. No separate submission step is needed.

    <Warning>
      Set `fillDeadline` well before `expires`. The solver must deliver before `fillDeadline`; `expires` is the final deadline after which you can claim a refund if the order wasn't filled.
    </Warning>
  </Step>

  <Step title="Track the order">
    Poll `GET /orders/status` until the order reaches a terminal state. Use the `onChainOrderId` from the `Open` event.

    ```ts TypeScript theme={"system"}
    const trackOrder = async (onChainOrderId: string) => {
      let status: string;
      do {
        const res = await fetch(
          `https://order.li.fi/orders/status?onChainOrderId=${onChainOrderId}`
        );
        const data = await res.json();
        status = data.meta.orderStatus;
        console.log(`Status: ${status}`);

        if (status !== 'Settled' && status !== 'Expired') {
          await new Promise(r => setTimeout(r, 3000));
        }
      } while (status !== 'Settled' && status !== 'Expired');

      return status;
    };

    await trackOrder(orderId);
    ```

    | Status      | Meaning                                                    |
    | ----------- | ---------------------------------------------------------- |
    | `Open`      | Order registered on-chain, tokens locked in escrow         |
    | `Signed`    | Order signed and available for solver pickup               |
    | `Delivered` | Solver has delivered assets on the destination chain       |
    | `Settled`   | Proof verified, locked funds released to solver. Complete. |

    Once `Settled`, the user has received USDC on Arbitrum and the solver has been paid from escrow.
  </Step>
</Steps>

***

## What Just Happened

1. **Requested a quote.** The order server returned pricing from its solver network based on your input/output pair.
2. **Approved tokens.** The escrow contract was authorized to transfer your USDC.
3. **Opened the order.** Tokens were locked in escrow and the intent was broadcast to solvers via the `Open` event.
4. **Solver delivered.** A solver fulfilled the order by delivering USDC on Arbitrum.
5. **Settlement completed.** The oracle verified delivery and the escrow released the locked funds to the solver.

***

## Next Steps

<CardGroup cols={2}>
  <Card title="API Overview" icon="code" href="/lifi-intents/intents-api/api-overview">
    Full endpoint reference, base URLs, and authentication
  </Card>

  <Card title="Request a Quote" icon="bolt" href="/lifi-intents/intents-api/request-quote">
    Exact-input, exact-output, and exclusive quote details
  </Card>

  <Card title="Compact Orders" icon="bolt" href="/lifi-intents/intents-api/compact-orders">
    Off-chain gasless order submission via The Compact
  </Card>

  <Card title="Track Order Status" icon="signal" href="/lifi-intents/intents-api/track-status">
    On-chain events and order server status polling
  </Card>
</CardGroup>
