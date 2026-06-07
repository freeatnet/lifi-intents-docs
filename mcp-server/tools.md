> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Available Tools

> Complete reference for all tools exposed by the LI.FI Intents MCP Server

The LI.FI Intents MCP Server exposes tools organized into two categories. Without an API key, only integrator tools are available. With a solver API key, all tools are enabled.

## Integrator Tools

These tools are available to everyone — no authentication required.

### get-supported-routes

Discover available chain + token pairs with fees and activity status. Use this to verify route availability before requesting a quote.

| Parameter | Required | Description                  |
| --------- | -------- | ---------------------------- |
| *none*    | —        | Returns all supported routes |

### request-quote

Get pricing for a cross-chain swap. Handles EIP-7930 encoding internally — you can use plain `0x` addresses and human-readable amounts.

| Parameter     | Required | Description                         |
| ------------- | -------- | ----------------------------------- |
| `fromChain`   | Yes      | Source chain ID or name             |
| `toChain`     | Yes      | Destination chain ID or name        |
| `fromToken`   | Yes      | Source token address or symbol      |
| `toToken`     | Yes      | Destination token address or symbol |
| `fromAmount`  | Yes      | Amount to swap (human-readable)     |
| `fromAddress` | Yes      | Sender wallet address               |

<Tip>
  The response includes a guided `message` suggesting the next tool to call — typically `prepare-order`.
</Tip>

### prepare-order

Construct a `StandardOrder` from a quote, optionally sign it via EIP-712. Returns an unsigned order structure if the server has no signer key.

| Parameter | Required | Description                              |
| --------- | -------- | ---------------------------------------- |
| `quoteId` | Yes      | Quote ID from a `request-quote` response |

### submit-order

Submit a signed order to the LI.FI order server.

| Parameter          | Required | Description                              |
| ------------------ | -------- | ---------------------------------------- |
| `order`            | Yes      | The order object from `prepare-order`    |
| `sponsorSignature` | Yes      | EIP-712 signature from the order sponsor |

<Note>
  Solver coverage varies — submitted orders may not be picked up immediately or at all depending on available solvers for the route.
</Note>

### track-order

Check order status, solver address, and transaction hashes. Status progresses: `Submitted -> Open -> Signed -> Delivered -> Settled`.

| Parameter | Required | Description       |
| --------- | -------- | ----------------- |
| `orderId` | Yes      | Order ID to track |

### list-orders

List orders with filters for status, user, chain, and pagination.

| Parameter | Required | Description                |
| --------- | -------- | -------------------------- |
| `status`  | No       | Filter by order status     |
| `user`    | No       | Filter by user address     |
| `chain`   | No       | Filter by chain ID         |
| `limit`   | No       | Number of results per page |
| `offset`  | No       | Pagination offset          |

## Solver Tools

These tools require a valid solver API key passed via the `x-api-key` header.

### get-solver-identities

View registered solver addresses and reputation tracking data.

| Parameter | Required | Description                                     |
| --------- | -------- | ----------------------------------------------- |
| *none*    | —        | Returns identities for the authenticated solver |

### get-quote-inventory

View uploaded quotes for a specific route. Flags expired quotes that need refreshing.

| Parameter   | Required | Description               |
| ----------- | -------- | ------------------------- |
| `fromChain` | Yes      | Source chain ID           |
| `toChain`   | Yes      | Destination chain ID      |
| `fromToken` | No       | Source token address      |
| `toToken`   | No       | Destination token address |

### submit-standing-quotes

Submit or update pricing quotes in bulk — up to 200K quotes per call.

| Parameter | Required | Description                                          |
| --------- | -------- | ---------------------------------------------------- |
| `quotes`  | Yes      | Array of quote objects with route, price, and expiry |

### debug-order

Deep inspect an order's full lifecycle, including transaction hashes and solver info.

| Parameter | Required | Description       |
| --------- | -------- | ----------------- |
| `orderId` | Yes      | Order ID to debug |

### check-route-health

Composite diagnostics tool: checks route support + quote inventory + recent activity.

| Parameter   | Required | Description               |
| ----------- | -------- | ------------------------- |
| `fromChain` | Yes      | Source chain ID           |
| `toChain`   | Yes      | Destination chain ID      |
| `fromToken` | No       | Source token address      |
| `toToken`   | No       | Destination token address |

### unregister-solver

Remove a registered solver address. This is a destructive operation — the tool requires confirmation before proceeding.

| Parameter | Required | Description                  |
| --------- | -------- | ---------------------------- |
| `address` | Yes      | Wallet address to unregister |

<Warning>
  Unregistering a solver address removes it permanently from reputation tracking. This action cannot be undone.
</Warning>

## Coming Soon

These tools are planned but not yet fully supported.

### register-solver

Register a wallet address for reputation tracking. The registration flow requires a multi-step process: retrieve a registration message from the Order Server, sign it with your solver wallet (ECDSA or EIP-1271), then submit the signed message back.

| Parameter   | Required | Description                |
| ----------- | -------- | -------------------------- |
| `address`   | Yes      | Wallet address to register |
| `signature` | Yes      | Ownership proof signature  |
