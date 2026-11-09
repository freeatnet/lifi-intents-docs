> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Testing Integration

> Set up your solver identity, register addresses, and test your integration on testnet or mainnet before going live.

## Step 1: Account Creation & Identity

Currently, solver identity is managed through a standalone dashboard.

1. **Register:** Go to the Solver UI: [intents.li.fi](https://intents.li.fi/) (or [devintents.li.fi](https://devintents.li.fi/) for testnet).
2. **Set Identity:** Choose a **Solver Name**. This is a permanent identifier used for your API keys and reputation tracking.
3. **Generate API Keys:** Create an API key in the dashboard. You will need this to submit quotes to the [`/quotes/submit`](https://order.li.fi/docs#tag/solver-api/post/quotes/submit) endpoint.

***

## Step 2: Address Registration

You must prove ownership of the wallet addresses you will use to fill intents.

1. **Fetch Message:** Get a one-time registration message from `GET /api/v1/solver/register/message`.
2. **Sign Message:** Sign it with your solver account's key — ECDSA or EIP-1271 on EVM, ed25519 on Solana.
3. **Submit Pairing:** Send the account, chain, message, and signature to [`POST /api/v1/solver/register`](https://order.li.fi/docs#tag/solver-api-v1/post/api/v1/solver/register).

See [Registering Accounts](https://docs.li.fi/lifi-intents/for-solvers/quoting.md#registering-accounts) for code examples.

> [!NOTE]
>
> Each address can only be tied to one API key. If your solver cannot sign messages, the LI.FI team can add a solver address for you.

***

## Step 3: Prove execution on mainnet with `exclusiveFor`

Before being whitelisted for the LI.FI API, you must prove your execution logic works on mainnet. You can test using either the Lintent UI or the LI.FI Intents MCP Server — both are equally valid approaches.

> [tab] Lintent UI
>
> ### Manual Testing via Lintent
>
> 1. **Open Lintent:** Go to [lintent.org](https://lintent.org/).
> 2. **Configure Intent:** Set up a swap/bridge intent. Use "Escrow" input instead of "The Compact".
> 3. **Set Exclusivity:** Manually enter your registered solver address into the `exclusiveFor` field. This ensures only your solver can fill this specific intent.
> 4. **Execute:** Submit the intent and verify that your solver receives the resulting order, fills it, and settles.
>
> ![Lintent UI showing the exclusiveFor field](https://mintcdn.com/lifi/6aLVXAyob7eMY6iC/images/intents/lintent-exclusivefor.png?fit=max\&auto=format\&n=6aLVXAyob7eMY6iC\&q=85\&s=80a4ab6053af124c5ab6aeb18dfa7bde)
>
> Lintent UI with the exclusiveFor field set to your solver address

> [tab] MCP Server (EVM only)
>
> ### Programmatic Testing via MCP
>
> The [LI.FI Intents MCP Server](https://docs.li.fi/lifi-intents/mcp-server/overview.md) can prepare and submit EVM test orders. Its `prepare-order` and `submit-order` flow does not open Solana or Tron orders.
>
> Use the same `exclusiveFor` approach, but driven by AI tools instead of a web UI:
>
> 1. Connect the MCP server to your AI tool (see [Installation](https://docs.li.fi/lifi-intents/mcp-server/installation.md))
>
> 2. Use `request-quote` to get pricing for your test swap
>
> 3. Use `prepare-order` and `submit-order` to issue the intent
>
> 4. Monitor with `track-order` and verify your solver completes the fill
>
> > [card] [MCP Server Overview](https://docs.li.fi/lifi-intents/mcp-server/overview.md)
> >
> > Quickstart, architecture, and environments
>
> > [card] [Installation](https://docs.li.fi/lifi-intents/mcp-server/installation.md)
> >
> > Setup for Claude, Cursor, VS Code, and more
>
> > [card] [Available Tools](https://docs.li.fi/lifi-intents/mcp-server/tools.md)
> >
> > Full reference for all tools
>
> > [card] [Examples](https://docs.li.fi/lifi-intents/mcp-server/examples.md)
> >
> > Workflows, prompts, and settlement flow

### Testing a Solana order

For Solana, request a V1 quote, add the returned compute-unit limit, replace the dummy blockhash, and sign and send the rebuilt transaction. The resulting `OpenEvent` is ingested and broadcast to your solver. See [Opening on Solana](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md#opening-on-solana) for the transaction code.

For a Solana-origin test, set `exclusiveFor` to the base58 Solana public key that will sign `finalise`, including when the output is on EVM. Complete the entire solver path—not only order ingestion—using the [Solana execution checklist](https://docs.li.fi/lifi-intents/for-solvers/settlement.md#solana-execution-checklist): fill, prove or receive the attestation, and finalize before expiry.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.