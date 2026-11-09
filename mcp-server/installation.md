> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Installation

> Setup instructions for connecting the LI.FI Intents MCP Server to your AI tool

The fastest way to get started is connecting to the **hosted server**. No installation required. Below are setup instructions for each supported AI tool.

## Claude Desktop

Add to your Claude Desktop config:

* **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
* **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`
* **Linux**: `~/.config/Claude/claude_desktop_config.json`

```json theme={"system"}
{
  "mcpServers": {
    "lifi-intents": {
      "type": "http",
      "url": "https://intents-mcp.li.fi/mcp"
    }
  }
}
```

With a solver API key for full access:

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

Restart Claude Desktop after saving.

## Claude Code

Add via CLI:

```bash theme={"system"}
claude mcp add lifi-intents --transport http --url https://intents-mcp.li.fi/mcp
```

Or add a `.mcp.json` file at your project root (shareable via git):

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

Use `https://intents-mcp-testnet.li.fi/mcp` for testnet. Run `/mcp` to connect.

## Cursor

Add to `.cursor/mcp.json` (project-level) or `~/.cursor/mcp.json` (global):

```json theme={"system"}
{
  "mcpServers": {
    "lifi-intents": {
      "url": "https://intents-mcp.li.fi/mcp",
      "headers": {
        "x-api-key": "your-solver-api-key"
      }
    }
  }
}
```

## Windsurf

Add to `~/.codeium/windsurf/mcp_config.json`:

```json theme={"system"}
{
  "mcpServers": {
    "lifi-intents": {
      "serverUrl": "https://intents-mcp.li.fi/mcp",
      "headers": {
        "x-api-key": "your-solver-api-key"
      }
    }
  }
}
```

> [!NOTE]
>
> Windsurf uses `serverUrl` instead of `url`.

## VS Code (GitHub Copilot)

Add to `.vscode/mcp.json` in your project:

```json theme={"system"}
{
  "inputs": [
    {
      "id": "solver-api-key",
      "type": "promptString",
      "description": "LI.FI Solver API Key",
      "password": true
    }
  ],
  "servers": {
    "lifi-intents": {
      "type": "http",
      "url": "https://intents-mcp.li.fi/mcp",
      "headers": {
        "x-api-key": "${input:solver-api-key}"
      }
    }
  }
}
```

## Signing Orders

**The hosted server never holds private keys.** You must sign orders yourself.

1. `request-quote` — get a quote for a cross-chain swap
2. `prepare-order` — returns an **unsigned** order structure
3. **Sign the order** using EIP-712 typed data signing with your wallet (e.g. viem, ethers.js)
4. `submit-order` — submit the signed order to the order server

> [!NOTE]
>
> All orders require gas. The signer pays gas on the source chain to initiate the cross-chain escrow.

## Auto-Signing (Local Server)

If you want the server to sign orders on your behalf, you can run the MCP server locally with a signer key:

1. Set up a `.env` file with your private key:
   ```
   SIGNER_PRIVATE_KEY=0xYourPrivateKeyHere
   ```
2. Start the local server — it will use the key to automatically sign orders during `prepare-order`
3. Connect your MCP client to `http://localhost:3000/mcp` instead of the hosted URL

> [!WARNING]
>
> Never share your private key or commit it to version control. The local server should only be used in trusted environments.

## Testing Your Setup

You can verify your setup by asking your AI tool a simple question like:

> "What cross-chain routes are available on LI.FI Intents?"

If the MCP server is connected correctly, the tool will call `get-supported-routes` and return a list of available chain + token pairs.

To verify solver tools are enabled, try:

> "Show my registered solver identities"

If your API key is configured correctly, the tool will call `get-solver-identities` and return your registered addresses.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.