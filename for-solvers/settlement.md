> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Settling Orders

> LI.FI Intents streamlines finalization and settlement, closing the loop on cross-chain orders and giving you confidence and control over the entire lifecycle.

To settle LI.FI Intents orders, up to 2 transactions may be required:

1. Submission of message validation
2. Finalizing the order on the resource lock.

## Oracle Validation

For instructions on how to relay proof for oracle systems, please refer to their section in [validation](/lifi-intents/architecture/oracle-systems).

## Finalizing Orders

Once the  `OutputProven` event has been observed, the output has been validated and `finalise` can be called.

```
event OutputProven(uint256 chainid, bytes32 remoteIdentifier, bytes32 application, bytes32 payloadHash);
```

Depending on the settler used, you either have to call a finalise function with a different abi. `InputSettlerCompact` uses a signature required finalise function where `InputSettlerEscrow` does not.

For the `InputSettlerCompact` submit the `StandardOrder`, the attached signatures as `bytes.concat(sponsor, allocator)`, the timestamps of the fills (which can be read from the `OutputFilled` event), and the solver's identifier. If there are multiple outputs, always index lists by their position in the order. Then, finalize can be called. The flow is the same for `InputSettlerEscrow` except no signatures are provided.

```solidity theme={"system"}
struct SolveParams(
    uint32 timestamp;
    bytes32 solver;
)
// For the Compact Input Settler
function finalise(
    StandardOrder calldata order,
    bytes calldata signatures,
    SolveParams[] calldata solveParams,
    bytes32 destination,
    bytes calldata call
) external;
// Or for the Escrow Input Settler
function finalise(
    StandardOrder calldata order,
    SolveParams[] calldata solveParams,
    bytes32 destination,
    bytes calldata call
) external;
```

<Note>
  If an order contains multiple outputs and two solvers filled different outputs, then the solver of the first output is the canonical solver.
</Note>

### Signature Finalise

The solver must be the caller of `finalise`, as they can redirect the funds and modify the optional callback. For some solvers, this is not be possible, i.e. when the solver is set to a pool. In that case, `finaliseWithSignature` can be used with an EIP-712 `AllowOpen` signature:

```solidity theme={"system"}
struct AllowOpen {
    bytes32 orderId;
    bytes32 destination;
    bytes call;
}
```

This allows anyone with the signature to call the Input Settler finalise function with the signed destination and optionally call.

### Callback

Some solvers may use callbacks for configuring received funds. The `call` parameter of `finalise[WithSignature]` forwards the provided payload to `destination` via `orderFinalised`:

```solidity theme={"system"}
function orderFinalised(
    uint256[2][] calldata inputs,
    bytes calldata call
) external
```

It is required that the `orderFinalised` call does not fail.

For a same chain intent, `orderFinalised` is executed after inputs are paid to `destination` but before outputs proofs are validate. This allows solvers to collect intent inputs before filling the intents in an atomic transaction.
