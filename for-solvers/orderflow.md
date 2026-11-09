> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Collecting Orders

> Receive orders over WebSocket or on-chain events and validate them before filling.

The order server takes intents (orders) from users and broadcasts them to solvers. It:

1. Collects user intents from various sources.
2. Broadcasts these intents to connected solvers in real-time.
3. Tracks the on-chain status of all transactions.

> [!NOTE]
>
> We recommend the order server for its built-in safety checks, but *for on-chain orders* it **is entirely optional**. Not every intent reaches the chain, though — [lintent.org](https://lintent.org), for one, does not emit intents on-chain. And any intent you read off the chain yourself, you must hydrate and validate very thoroughly.

## Order subscription

You can collect orders in two ways: a WebSocket subscription that pushes orders to you, or a GET endpoint you poll. We recommend the WebSocket subscription. Both methods can be found in the [lintent.org](https://lintent.org) demo [implementation](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/api.ts#L75-L114).

If you use the GET endpoint too aggressively, you may be throttled or temporarily timed out. You can find swagger documentation on api interfaces on [https://order-dev.li.fi/docs](https://order-dev.li.fi/docs).

> [!NOTE]
>
> You may receive the same order more than once through the WebSocket interface. It happens each time an intent is ingested from a source of truth, and the later copies can carry extra context. Use the `orderId` to identify an intent, and keep the extra context when a later copy brings it.

### Order Server events

The WebSocket host is `wss://order.li.fi` (production) or `wss://order-dev.li.fi` (development). Messages are JSON frames of the form `{ "event": string, "data": ... }`:

* **`user:vm-order-submit`** — a new order is available. The data is returned similarly to how it was [submitted](https://order-dev.li.fi/docs#:~:text=SubmitOrderDto): the full order plus metadata (order id, status, timestamps, quote id).
* **`intent:transaction-observed`** — lifecycle updates when an order becomes `Delivered`, `Finalised`, or `Refunded`.
* **Heartbeat** — the server sends `{"event": "ping"}` every 5 seconds; reply with `{"event": "pong"}` within 10 seconds or the socket is closed. This is used to disconnect misbehaving clients and for debugging purposes. You can find an example [implementation here](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/api.ts#L89-L95).

## Order Validation

LI.FI Intents allows highly customizable orders, so you must add your own validation to confirm you support each order relayed to you.

Below, whitelisted means **trusted and validated by you**. When a token *must* be whitelisted, it means **you** have vetted the token. If a validation layer is whitelisted, it means **you** have vetted the validation layer, etc. Whitelisted does not mean permissioned by a central entity.

This list aims to cover every validation a solver must implement. It may look long, but you have likely already implemented most of these checks.

### General Validation

1. `fillDeadline`: Ensure you have sufficient time to fill the order:
   * Time to fill on the destination chain, including potential source chain finality.
2. `expires`: Ensure you have enough time to fill, relay, and claim the order:
   * Time to fill on the destination chain, including potential source chain finality.
   * Time to send message validation proof to the input chain.
   * Time to claim the order after validation has been delivered.
3. Validation layer: Ensure you support submitting proofs (and, if not automatic, also relaying) to the validation layer. Additionally, the `inputOracle` and `output.oracle` must be of the same validation layer.
4. Ensure you have whitelisted every `input` token. If one input token is malicious, the order may be unclaimable. Additionally, for blacklistable tokens like USDC, ensure you are not on a blacklist.
   * If an order has a 0 input or output of a token you have not whitelisted, the order may not be fillable. Be careful about filling orders containing unfamiliar tokens.
5. For each output:
   1. `output.chainId` is whitelisted.
   2. `output.oracle` and `inputOracle` are correctly configured regarding `originChainId` and `output.chainId`. On-chain configs are immutable, so this can be done once for each chain pair.
   3. `output.settler` is whitelisted.
   4. `output.context` is decodable, and the order type is supported and compatible with `output.settler`.
   5. `output.token` is whitelisted. Additionally, for blacklistable tokens like USDC, ensure that neither you nor the recipient is on a blacklist.
      * If an order has a 0 input or output of a token you have not whitelisted, the order may not be fillable. Be careful about filling orders containing unfamiliar tokens.
   6. You have sufficient tokens for `output.amount`.
   7. If the output has `calldata`, ensure you can execute it and other outputs atomically. For outputs on different chains, you may have to whitelist recipients if there is `calldata`.
      * On OP-chains, [CrossL2Inbox](https://specs.optimism.io/interop/predeploys.html#validatemessage) needs to be blacklisted in the entire call tree. If similar contracts exist on another chain, they also need to be blacklisted.
   8. Neither `call.length` nor `context.length` is more than 65,535 bytes long.
   9. Validate the context depending on order type. For Bitcoin specifically, ensure the encoded multiplier is relative to the Bitcoin value.
6. If the order has multiple outputs, ensure you can fill all outputs and the first output is set to your solver identifier. If all outputs are on the same chain, a single `fillOrderOutputs` call fills them all as a protective measure.
7. If the InputSettler has any fees, check for an imminent fee change.

> [!NOTE]
>
> On EVM and Tron, input tokens are encoded as `uint256`; the first 12 bytes may hold a lock tag and the address is the last 20 bytes. A Solana order instead carries one `MandateInput { token: Pubkey, amount: u64 }`. In Solana feed messages, top-level addresses render as base58, while output `oracle`, `settler`, `token`, and `recipient` values remain 32-byte fields.

### Escrow Validation

1. Ensure no input tokens are fee on transfer.
2. Ensure every `inputs` token has no set bits in its upper 12 bytes.
3. Ensure the order has already been opened, either through the `event Open` or `function orderStatus` OR you are opening the order through `function openFor` before filling the order.

### Solana Validation

Solana orders are escrow-only, so the Compact checks below do not apply. On top of the general validation, check:

1. **Identity fields.** Re-derive every 32-byte identity in the order — `outputs[].settler`, `outputs[].oracle`, `inputOracle` — from the published program IDs and seeds (or pin the published addresses as golden values) and check them against the [identity matrix](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#which-identity-goes-in-which-field). **Reject on any mismatch.**
2. **`u64` bounds.** `MandateOutput.amount` and Dutch-auction `slope` values above `u64::MAX` make an output unfillable on Solana ([Auctions](https://docs.li.fi/lifi-intents/for-solvers/auctions.md#solana-encoding-notes)).
3. **Callbacks.** The default Solana output settler requires `callback_data` to be empty and rejects fills otherwise.
4. **Recipient accounts.** For SPL outputs, `recipient_token_account.owner` must equal `recipient`; you may pay ATA rent when the recipient has no token account yet.
5. **Order size.** A single-output order with no calldata and a standard context is always safe to finalise. Anything beyond that requires thorough validation against the finalise transaction limit — see [Order size](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md#order-size) for the exact bound.

### Compact Validation

1. Ensure every `input` token uses the same `allocatorId`. If the `lockId` values differ, check the lock expiry for every one of them; if they are all equal, one expiry check is enough.
2. Ensure the potential `reset period` for a resource lock extends beyond the `expiry` AND there is no active withdrawal.
3. Ensure the `allocatorID` is whitelisted. The allocator can block claims from processing (by withdrawing signatures or reusing nonces.)
   * The allocatorID is part of the `lock tag` for `inputs` (first 12 bytes).
   * Optionally, ensure the user has sufficient tokens. This should have been validated by the allocator, though.
4. Ensure the user provides an ECDSA signature **or** they have a whitelisted [emissary](https://docs.li.fi/lifi-intents/knowledge-database/glossary.md#emissary) registered **or** you register the claim on-chain before filling the claim.
   * Note: If any of these actions are taken, the signature can be assumed to be trusted.
   * Emissary signatures cannot be registered on-chain. The emissary must be trusted not to redact a signature.
5. Validate the `allocatorData`. You may have to do an on-chain call.
6. Validate that the allocator `nonce` has not been spent previously by any user. The Order nonce is not a user nonce.

### Signature Validation

The `StandardOrder` is what you pass to every function on the input chain. Once hydrated with a signature, it also lets you check that an order is valid.

The `StandardOrder` struct will be signed and stored as a witness in the appropriate lock/claim structure. For TheCompact, this is:

```solidity theme={"system"}
struct BatchCompact {
    address arbiter; // Associated settlement contract
    address sponsor; // StandardOrder.user
    uint256 nonce; // StandardOrder.nonce
    uint256 expires; // StandardOrder.expires
    uint256[2][] idsAndAmounts; // StandardOrder.inputs
    Mandate mandate;
}

struct Mandate {
    uint32 fillDeadline; // StandardOrder.fillDeadline
    address inputOracle; // StandardOrder.inputOracle
    MandateOutput[] outputs; // StandardOrder.outputs
}
```

To validate an order, ensure that the sponsor and allocator signatures are valid for this EIP-712 signed structure.

#### On-chain Order Broadcast

When intents are registered on-chain, they can be broadcasted on the Input Settlement contract through the [`IntentRegistered`](https://github.com/catalystsystem/catalyst-intent/blob/27ce0972c150ed113f66ae91069eb953f23d920b/src/input/compact/InputSettlerCompactLIFI.sol#L67) event:

```solidity theme={"system"}
event IntentRegistered(bytes32 indexed orderId, StandardOrder order);
```

> [!NOTE]
>
> When collecting permissionless orders, ensure they are properly validated and co-signed. The order server helps with validation, but on-chain, anyone can emit a fraudulent message.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.