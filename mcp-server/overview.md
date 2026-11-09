> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Intents MCP Server

> Connect AI tools to LI.FI Intents cross-chain escrow orders and solver operations via the Model Context Protocol

The LI.FI Intents MCP Server connects to the [LI.FI Intents API](https://docs.li.fi/lifi-intents/introduction.md) so AI tools can manage cross-chain escrow orders and run solver operations over the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/).

Built for **solver operators**: instead of clicking through the [Lintent.org](https://lintent.org) UI, you manage quotes, drive orders through their lifecycle, and diagnose routes in code.

> [!WARNING]
>
> Unlike the general [LI.FI MCP server](https://docs.li.fi/mcp-server/overview.md) (read-only quotes and routing), this server performs **write operations** — it submits signed orders and manages standing quotes (solver registration is planned). The hosted server never holds private keys.

## Quickstart

Add the hosted LI.FI Intents MCP server to your AI tool. No installation required:

```json theme={"system"}
{
  "mcpServers": {
    "lifi-intents": {
      "type": "http",
      "url": "https://intents-mcp.li.fi/mcp",
      "headers": {
        "x-api-key": "your-solver-api-key"
      }
    }
  }
}
```

Paste this into your MCP client config and start querying intents data. See [Installation](https://docs.li.fi/lifi-intents/mcp-server/installation.md) for tool-specific setup instructions.

The `x-api-key` header is only required for solver tooling. If you have a Solver API key, set it as `x-api-key` in `headers`; otherwise remove the field.

## How It Works

The MCP server wraps the LI.FI Intents REST API into MCP-compatible tools that AI agents can call directly. Instead of constructing HTTP requests, your AI tool discovers and calls structured tools like `request-quote`, `prepare-order`, and `submit-standing-quotes`.

```
AI Tool (Claude, Cursor, etc.)
    |
    v
MCP Protocol
    |
    v
LI.FI Intents MCP Server (https://intents-mcp.li.fi/mcp)
    |
    v
LI.FI Order Server (https://order.li.fi)
    |
    v
Cross-chain escrow settlement via Polymer oracle
```

## Example Workflow

A typical cross-chain swap via the MCP server follows this flow:

```
1. get-supported-routes          # Discover available chain + token pairs
2. request-quote                 # Get pricing (chain names, symbols, or IDs all work)
3. prepare-order                 # Build order structure (unsigned)
4. (external) Sign the order using EIP-712 typed data signing
5. submit-order                  # Submit signed order to the order server
6. track-order                   # Monitor: Signed -> Delivered -> Settled
```

## API Key Configuration

Without an API key, only **integrator tools** are available (read-only operations). With an API key, solver tools are also enabled, including `submit-standing-quotes`, `debug-order`, and `check-route-health`.

1. Go to **[intents.li.fi](https://intents.li.fi)** (mainnet) or **[devintents.li.fi](https://devintents.li.fi)** (testnet)
2. Connect your wallet and register as a solver
3. Copy your API key from the dashboard
4. Add it in `"headers"` as `x-api-key` in your MCP config

> [card] [Get a Solver API Key](https://intents.li.fi)
>
> Register at intents.li.fi to get your solver API key

## Environments

| Environment | MCP Server URL | Order Server | Chains |
| - | - | - | - |
| **Mainnet** | `https://intents-mcp.li.fi/mcp` | `https://order.li.fi` | Ethereum, Base, Arbitrum, Optimism, + more |
| **Testnet** | `https://intents-mcp-testnet.li.fi/mcp` | `https://order-dev.li.fi` | Ethereum Sepolia, Base Sepolia, Optimism Sepolia, Arbitrum Sepolia |

## Contract Addresses

Shared across testnet and mainnet:

| Contract | Address |
| - | - |
| **Escrow Settler** | `0x00fC00edbe7C003b006f870068c548940000223e` |
| **Output Settler** (`OutputSettlerSimple`) | `0x75220B7600c300005038432a0000f308e0000068` |

**Polymer Oracle** differs by environment:

| Environment | Oracle Address |
| - | - |
| Testnet | `0xC401b53377b8A71A7cEB820e6a4dC53832343a90` |
| Mainnet | `0x008C3800F3Ad9b3B662d002E90Cc00000000eE17` |

## Next Steps

> [card] [Installation](https://docs.li.fi/lifi-intents/mcp-server/installation.md)
>
> Setup instructions for Claude, Cursor, Windsurf, VS Code, and more

> [card] [Available Tools](https://docs.li.fi/lifi-intents/mcp-server/tools.md)
>
> Complete reference for all integrator and solver tools

> [card] [Examples](https://docs.li.fi/lifi-intents/mcp-server/examples.md)
>
> Example workflows, prompts, and settlement flow


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.