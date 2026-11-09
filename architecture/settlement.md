> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Settlement

> How input and output settlement works in LI.FI Intents, covering escrow, Compact, and output settler contracts.

Settlement is the process of escrowing user funds (input settlement) and recording solver delivery (output settlement). This page covers both sides.

***

## Input settlement

LI.FI Intents supports two single-chain input settlers:

* [**InputSettlerEscrow**](https://github.com/openintentsframework/oif-contracts/blob/main/src/input/escrow/InputSettlerEscrow.sol). Standard escrow that associates tokens with their intent.
* [**InputSettlerCompact**](https://github.com/openintentsframework/oif-contracts/blob/main/src/input/compact/InputSettlerCompact.sol). Long-lived deposits via The Compact resource lock.

[Multi-chain input settlers](https://github.com/openintentsframework/oif-contracts/pull/49) are in progress.

The Compact supports fill-first flows through resource locks. The Escrow settler uses traditional flows where the escrow must be opened before fills.

### StandardOrder

Single-chain input settlers use the following order structure:

```solidity theme={"system"}
struct StandardOrder {
    address user;
    uint256 nonce;
    uint256 originChainId;
    uint32 expires;
    uint32 fillDeadline;
    address inputOracle;
    uint256[2][] inputs;
    MandateOutput[] outputs;
}
```

### MandateOutput

```solidity theme={"system"}
struct MandateOutput {
    bytes32 oracle;
    bytes32 settler;
    uint256 chainId;
    bytes32 token;
    uint256 amount;
    bytes32 recipient;
    bytes call;
    bytes context;
}
```

### Finalization

To claim input funds after filling an order, call one of:

1. **`finalise`** — the solver calls this and chooses where the input assets are sent and whether to make an external call. The Escrow and Compact `finalise` interfaces differ, to optimize for gas.
2. **`finaliseWithSignature`** — anyone can call this if they hold an `AllowOpen` signature in which the solver authorized the payout destination and call details.

### Compact Registration

A Compact intent is carried in the `BatchCompact` payload below and signed as part of a `BatchClaim`, using The Compact's EIP-712 domain separator:

```solidity theme={"system"}
struct BatchCompact {
    address arbiter;
    address sponsor;
    uint256 nonce;
    uint256 expires;
    uint256[2][] idsAndAmounts;
    Mandate mandate;
}

struct Mandate {
    uint32 fillDeadline;
    address inputOracle;
    MandateOutput[] outputs;
}
```

Alternatively, intents can be registered on-chain by the sponsor or by someone paying for the claim on their behalf.

### Escrow Deposit Methods

Escrow intents can be deposited in several ways:

1. **Direct open** by the owner through ERC-7683 `function open(bytes)`. Emits `event Open(bytes32 indexed orderId, bytes order)`.
2. **ERC-3009** with the orderId as nonce. A signature per input is required.
3. **Permit2** with an EIP-712 signed `PermitBatchWitnessTransferFrom`.

### Integration Examples

* Smart contract registration: [`RegisterIntentLib.sol`](https://github.com/catalystsystem/catalyst-intent/blob/27ce0972c150ed113f66ae91069eb953f23d920b/src/libs/RegisterIntentLib.sol#L100-L131)
* UI signing (Batch Compact): [lintent.org demo](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/lifiintent/tx.ts#L144)
* UI deposit and registration: [lintent.org demo](https://github.com/lifinance/lintent/blob/a4aa78cd058cade732b73d83aa2843dd4e9ea24d/src/lib/utils/lifiintent/tx.ts#L199)

***

## Output settlement

Currently one output settler exists: [`OutputSettlerSimple.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/output/simple/OutputSettlerSimple.sol). It supports four order types: simple limit orders, Dutch auctions, exclusive limit orders, and exclusive Dutch auctions.

### Configuring Auction Type

The `context` field in `MandateOutput` sets the auction type:

```solidity theme={"system"}
// Simple limit order (no auction)
context = "0x" or "0x00";

// Dutch auction
context = abi.encodePacked(bytes1(0x01), uint32(startTime), uint32(stopTime), uint256(slope));
```

For Dutch auctions with multiple outputs, only the first output functions as an auction. The rest resolve to their worst price, because solvers only compete on the first output (winning it wins the entire order).

For more details on auction mechanics, see [Auctions](https://docs.li.fi/lifi-intents/for-solvers/auctions.md).

***

## Next Steps

> [card] [System Overview](https://docs.li.fi/lifi-intents/architecture/overview.md)
>
> Full architecture and contract addresses

> [card] [Oracle Systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md)
>
> Verification and proof systems

> [card] [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md)
>
> Order construction for integrators

> [card] [Solving LI.FI Intents](https://docs.li.fi/lifi-intents/for-solvers/intro.md#filling-intents)
>
> How solvers deliver outputs


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.