> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Intents MCP Server

> Connect AI tools to LI.FI Intents cross-chain escrow orders and solver operations via the Model Context Protocol

The LI.FI Intents MCP Server integrates with the [LI.FI Intents API](https://docs.li.fi/intents) to provide cross-chain escrow order management and solver operations via the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/).

Built for **solver operators** — replaces the manual [Lintent.org](https://lintent.org) UI workflow with programmatic access to quote management, order lifecycle, and route diagnostics.

<Warning>
  Unlike the general [LI.FI MCP server](/mcp-server/overview) (read-only quotes and routing), this server performs **write operations** — it submits signed orders, registers solvers, and manages standing quotes. The hosted server never holds private keys.
</Warning>

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

Paste this into your MCP client config and start querying intents data. See [Installation](/lifi-intents/mcp-server/installation) for tool-specific setup instructions.

The `x-api-key` header is only required for solver tooling. If you have a Solver API key, set it as `x-api-key` in `headers`; otherwise remove the field.

## How It Works

The MCP server wraps the LI.FI Intents REST API into MCP-compatible tools that AI agents can call directly. Instead of constructing HTTP requests, your AI tool discovers and invokes structured tools like `request-quote`, `prepare-order`, and `submit-standing-quotes`.

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
6. track-order                   # Monitor: Submitted -> Open -> Signed -> Delivered -> Settled
```

## API Key Configuration

Without an API key, only **integrator tools** are available (read-only operations). With an API key, solver tools are also enabled, including `submit-standing-quotes`, `debug-order`, and `check-route-health`.

1. Go to **[intents.li.fi](https://intents.li.fi)** (mainnet) or **[devintents.li.fi](https://devintents.li.fi)** (testnet)
2. Connect your wallet and register as a solver
3. Copy your API key from the dashboard
4. Add it in `"headers"` as `x-api-key` in your MCP config

<Card title="Get a Solver API Key" icon="key" href="https://intents.li.fi">
  Register at intents.li.fi to get your solver API key
</Card>

## Environments

| Environment | MCP Server URL                          | Order Server              | Chains                                                             |
| ----------- | --------------------------------------- | ------------------------- | ------------------------------------------------------------------ |
| **Mainnet** | `https://intents-mcp.li.fi/mcp`         | `https://order.li.fi`     | Ethereum, Base, Arbitrum, Optimism, + more                         |
| **Testnet** | `https://intents-mcp-testnet.li.fi/mcp` | `https://order-dev.li.fi` | Ethereum Sepolia, Base Sepolia, Optimism Sepolia, Arbitrum Sepolia |

## Contract Addresses

Shared across testnet and mainnet:

| Contract                       | Address                                      |
| ------------------------------ | -------------------------------------------- |
| **Escrow Settler**             | `0x000025c3226C00B2Cdc200005a1600509f4e00C0` |
| **Compact Settler**            | `0x0000000000cd5f7fDEc90a03a31F79E5Fbc6A9Cf` |
| **Multichain Escrow Settler**  | `0xb912b4c38ab54b94D45Ac001484dEBcbb519Bc2B` |
| **Multichain Compact Settler** | `0x1fccC0807F25A58eB531a0B5b4bf3dCE88808Ed7` |
| **CoinFiller**                 | `0x0000000000eC36B683C2E6AC89e9A75989C22a2e` |
| **The Compact**                | `0x00000000000000171ede64904551eeDF3C6C9788` |

**Polymer Oracle** differs by environment:

| Environment | Oracle Address                               |
| ----------- | -------------------------------------------- |
| Testnet     | `0xC401b53377b8A71A7cEB820e6a4dC53832343a90` |
| Mainnet     | `0x0000003E06000007A224AeE90052fA6bb46d43C9` |

## Next Steps

<CardGroup cols={2}>
  <Card title="Installation" icon="download" href="/lifi-intents/mcp-server/installation">
    Setup instructions for Claude, Cursor, Windsurf, VS Code, and more
  </Card>

  <Card title="Available Tools" icon="wrench" href="/lifi-intents/mcp-server/tools">
    Complete reference for all integrator and solver tools
  </Card>

  <Card title="Examples" icon="code" href="/lifi-intents/mcp-server/examples">
    Example workflows, prompts, and settlement flow
  </Card>

  <Card title="GitHub Repository" icon="github" href="https://github.com/lifinance/lifi-intents-mcp">
    Source code and self-hosting instructions
  </Card>
</CardGroup>
