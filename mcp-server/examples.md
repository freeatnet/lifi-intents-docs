> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Examples

> Example workflows, prompts, and end-to-end settlement flow for the LI.FI Intents MCP Server

## Example Workflows

### Cross-Chain Swap

A full order lifecycle from quote to settlement:

```
1. get-supported-routes          # Discover available routes
2. request-quote                 # Get pricing (chain names, symbols, or IDs all work)
3. prepare-order                 # Build order, auto-sign if key is configured
4. submit-order                  # Submit to order server
5. track-order                   # Monitor: Signed -> Delivered -> Settled
```

### Solver Operations

Managing your solver presence and diagnosing issues:

```
1. register-solver               # Register wallet for reputation tracking (coming soon — see Tools)
2. submit-standing-quotes        # Publish pricing for routes
3. get-quote-inventory           # Audit your published quotes
4. check-route-health            # Diagnose why orders aren't being filled
5. debug-order                   # Inspect a specific order's full lifecycle
```

***

## Example Prompts

These prompts work with any MCP-compatible AI tool connected to the LI.FI Intents MCP server.

### Discover Routes

**Prompt:** "What cross-chain routes are available on LI.FI Intents?"

**Tool called:** `get-supported-routes`

Returns available chain + token pairs with fees, activity status, and supported settlement methods.

### Get a Quote

**Prompt:** "Get me a quote for 5 USDC from Ethereum to Base for 0x42fB...749b"

**Tool called:** `request-quote` with chain names, token symbol, and recipient address. The server handles EIP-7930 encoding internally — no need to convert amounts to smallest units.

### Submit an Order

**Prompt:** "Prepare and submit an order from that quote"

**Tools called:**

1. `prepare-order` — builds the `StandardOrder` from the cached quote
2. The AI prompts you to sign the order using EIP-712
3. `submit-order` — submits the signed order to the order server

### Check Route Health

**Prompt:** "Check the health of the Ethereum Sepolia to Base Sepolia USDC route"

**Tool called:** `check-route-health` — checks several things at once: route support, quote inventory, and recent activity. Useful for diagnosing why orders aren't being filled.

### Debug an Order

**Prompt:** "Debug order intent\_YaY\_b3qxF3..."

**Tool called:** `debug-order` — returns the order's full lifecycle, transaction hashes on both chains, solver info, and settlement status.

***

## End-to-End Settlement Flow

To complete a full cross-chain swap (not just submit an order, but actually move tokens on-chain), you need to:

1. **Approve the escrow settler** to spend your tokens
2. **Request a quote, prepare, and submit** the order via MCP
3. **Deposit funds** into the escrow contract on-chain

Once deposited, a solver picks up the order, delivers tokens on the destination chain, and the settlement completes automatically.

### Step 1: Token Approval

Approve the escrow settler contract to spend your input token. On the supported EVM chains, the escrow settler uses the same address:

```
0x00fC00edbe7C003b006f870068c548940000223e
```

Using viem/TypeScript:

```typescript theme={"system"}
import { createWalletClient, http, parseAbi } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains"; // or baseSepolia for testnet

const account = privateKeyToAccount("0x<YOUR_PRIVATE_KEY>");
const client = createWalletClient({ account, chain: mainnet, transport: http("YOUR_RPC_URL") });

const tx = await client.writeContract({
  address: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", // token address (e.g. WETH on mainnet)
  abi: parseAbi(["function approve(address spender, uint256 amount) returns (bool)"]),
  functionName: "approve",
  args: [
    "0x00fC00edbe7C003b006f870068c548940000223e", // escrow settler
    BigInt("0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"), // max approval
  ],
});
```

### Step 2: Submit Order via MCP

Use the MCP tools in sequence:

```
1. request-quote    -> Get pricing (e.g., "0.001 WETH from Ethereum to Base")
2. prepare-order    -> Build + sign the order (auto-signs if SIGNER_PRIVATE_KEY is set locally)
3. submit-order     -> Submit to the LI.FI order server
```

After submission, the order status is `Signed` — but funds are not yet locked on-chain.

### Step 3: Deposit into Escrow

Call `open()` on the escrow settler contract to lock your tokens. This triggers a `transferFrom` using your prior approval.

```typescript theme={"system"}
import { encodeFunctionData, createWalletClient, http, parseAbi } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains";

const ESCROW_SETTLER = "0x00fC00edbe7C003b006f870068c548940000223e";

// The order object from prepare-order — pass the full encoded order bytes
const tx = await client.sendTransaction({
  to: ESCROW_SETTLER,
  data: encodeFunctionData({
    abi: parseAbi(["function open(bytes calldata order) external"]),
    functionName: "open",
    args: [orderBytes], // the encoded order from prepare-order
  }),
});
```

### What Happens Next

Once the escrow is funded:

1. **Solver sees the deposit** and delivers tokens on the destination chain
2. **Settlement** happens automatically via the Polymer oracle
3. **Order status** progresses: `Signed -> Delivered -> Settled`

Use `track-order` to monitor the full lifecycle. A typical mainnet settlement takes \~10 minutes.

### Testnet vs Mainnet

| | Testnet | Mainnet |
| - | - | - |
| **Chains** | Ethereum Sepolia, Base Sepolia, Optimism Sepolia, Arbitrum Sepolia | Ethereum, Base, Arbitrum, Optimism, + more |
| **Tokens** | Testnet USDC, testnet WETH | Real USDC, WETH, ETH, etc. |
| **Solver availability** | Limited — may not have active solvers | Active solvers filling orders |
| **RPC** | Public Sepolia RPCs work | Use Alchemy/Infura for reliability |
| **MCP URL** | `https://intents-mcp-testnet.li.fi/mcp` | `https://intents-mcp.li.fi/mcp` |


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.