> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Solving LI.FI Intents

> Learn how to receive, validate, fill, prove, and settle LI.FI intent orders.

## Solving Intents

LI.FI Intents separates order discovery, filling, proving, and settlement. Validate every order and every component address before committing inventory. The general flow is:

1. The user opens an escrow order on-chain or signs an EVM Compact order for submission to the LI.FI order server.

2. The order server validates and broadcasts the order. Compact orders also require the allocator's co-signature.

3. A solver submits the order's output to the output settlement contract, starting the oracle system.

   > [!NOTE]
   >
   > [Output Settlement](https://docs.li.fi/lifi-intents/architecture/settlement.md#output-settlement)
   > is denoted as the `output.settler`.

4. The proof is delivered to the input chain through the validation layer.

   > [!NOTE]
   >
   > [Oracle system](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md)
   > is denoted as the `inputOracle` and `output.oracle` in the order struct.

5. The solver submits the order to the input settlement contract, verifying the delivery and unlocking the associated input tokens.

### Order Types

For EVM Compact orders, you will meet an order in three forms, listed from least to most verbose:

1. `BatchClaim`: A signed intent that allows the input assets to be claimed.
2. `StandardOrder`: An on-chain definition containing enough information to convey the intent.
3. Order server response: a **hydrated** order — the `StandardOrder` plus the off-chain fields a solver needs, such as `InputSolver` and `signatures`.

Unless you are verifying Compact signatures, you won't interact with `BatchClaim`. On-chain interactions use a `StandardOrder`, while the order server hydrates it with the off-chain fields needed for a fill.

> [!NOTE]
>
> Solana also uses `StandardOrder`, but its Borsh structure carries exactly one input and uses Solana-native types. See [Solana vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md) for the structural and execution differences.

### Collecting Orders

LI.FI Intents offers two ways to receive orders:

1. **Recommended**: Connect to the order server via WebSocket.
2. **Alternative**: Monitor on-chain deposits through the deposit interface.

Most solvers should use the order server. But if you want the lowest latency, or you want order discovery to stay decentralized and permissionless, you can read some orders directly on-chain — though not every order is discoverable that way. The EVM `StandardOrder` representation is:

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

Where `uint256[2][] inputs === [uint256 tokenId, uint256 amount][]` and `MandateOutput` is:

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

For more information on collecting orders, [View Order Collection Guide →](https://docs.li.fi/lifi-intents/for-solvers/orderflow.md)

## Filling Intents

To fill an intent, you must execute every `MandateOutput`. Each one describes a single delivery completely on its own.

* `output.settler` defines that delivery together with its context — call it on the settler's interface. Currently, only one OutputSettler is implemented: [`OutputSettlerSimple.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/output/simple/OutputSettlerSimple.sol), which has two interfaces for filling:

```solidity theme={"system"}
/// @notice For implementing own batching functionality
function fill(
    uint32 fillDeadline,
    bytes32 orderId,
    MandateOutput calldata output,
    bytes32 proposedSolver
) external returns (bytes32 actualSolver);

/// @notice For batch filling entire orders.
function fillOrderOutputs(
    uint32 fillDeadline,
    bytes32 orderId,
    MandateOutput[] calldata outputs,
    bytes32 proposedSolver
) external;
```

On Solana, there is no batch fill: each output is filled with its own `fill` (or `native_fill` for SOL) instruction — see the [Solana program reference](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#fills). For a multi output intent, please ensure all outputs can be filled, ideally atomically.

The output-settlement architecture and interfaces are documented in [Settlement](https://docs.li.fi/lifi-intents/architecture/settlement.md#output-settlement).

## Validating Fills

After filling intents, the proof of fill has to be sent to the input chain. `inputOracle` and `output.oracle` must identify the two supported ends of the same oracle route, but they are not always the same address. On Solana, one field may hold a program ID while the other holds a state PDA. Proof validation is oracle-specific; see [Oracle System Architecture](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md#implemented-validation-interfaces).

## Settling Orders

Lastly, intents have to be settled before they expire. Settle by calling `finalise[withSignature]` on the designated InputSettler. The caller has to be the designated solver set on fill.

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

For more information about settling orders [View Order Settlement Guide →](https://docs.li.fi/lifi-intents/for-solvers/settlement.md)


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.