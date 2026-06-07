> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Oracle Systems

> LI.FI Intents lets you plug in custom validation and oracle layers, empowering you to choose the optimal trust and speed tradeoffs for your cross-chain use case.

Any validation layer can be added as long as it supports validating a payload from the output chain. LI.FI intent supports any oracle that the Open Intents Framework (OIF) supports. An OIF oracle validation layer needs to support the following:

1. A submission interface where solvers can submit their filled outputs. The submission interface should accept arbitrary packages for validation and then call the associated output settlement contract to check whether the payloads are valid. If *submission* is automatic by being capable of proving contract calls, events, or storage this requirement can be ignored.

   Oracle implementations can validate whether an output has been filled through the `IAttester` interface:

   ```solidity theme={"system"}
   interface IAttester {
     function hasAttested(
         bytes[] calldata payloads
     ) external view returns (bool);
   }
   ```

2. Implement the validation interfaces so attested (or filled) outputs can verify whether outputs have been filled:
   ```solidity theme={"system"}
     interface IInputOracle {
         function efficientRequireProven(
           bytes calldata proofSeries
       ) external view;
     }
   ```

Beware of the following when implementing oracle systems:

* Submission or receive interfaces do not have to be standardized; they need to be documented so solvers can implement it.
* Oracle systems can either have automatic relaying or manual relaying.
* Trust assumptions for the oracle system needs to be explicit, so intent issuers and solvers can make informed decisions.

For more documentation refer to the [OIF Contributing Guidelines](https://github.com/openintentsframework/oif-contracts/blob/main/CONTRIBUTING.md#oif-oracle-pr-requirements). These requirements specify how a proper OIF compatible oracle system should behave.

## Implemented Validation Interfaces

There are three types of validation interfaces:

1. **Self-serve**: Validation interfaces where submitting the payload generates an off-chain proof that must be collected and submitted on the input chain.
   * [Polymer](#polymer)
   * [Wormhole](#wormhole)

2. **Automatic**: Validation interfaces where submitting the payload automatically delivers the associated proof on the input chain.
   * [Hyperlane](#hyperlane)
   * [Chainlink CCIP](#chainlink-ccip)
   * [Axelar](#axelar)
   * [LayerZero](#layerzero)

3. **Same Chain**: Using the [Output Settlement](/lifi-intents/architecture/output-settlement) to produce an oracle proof for itself, skipping a dedicated oracle system.
   * [OutputSettlerSimple](#outputsettlersimple)

Choosing an oracle system is up to the intent issuer. Oracle security is directly linked to the payment of users' inputs to solvers, since users are paid directly by solvers. When issuing an intent, it is important that solvers are capable of understanding how to fill an intent using a specific oracle. Users can freely choose which oracle system they want to secure a specific intent, but solvers have to implement it for the intent to be solved.

When using the LI.FI intent order server, it will only support oracle systems that solvers are quoting for at a specific time.

<Note>
  **Speed & Price**\
  Some oracle systems are significantly faster than other oracle systems. While speed does not impact asset delivery for users, it matters for solvers. Choosing an oracle system with fast repayments results in cheaper intents, as solvers can rotate their capital faster.
</Note>

The following documentation may be outdated if the oracle network updates their documentation and does not notify the LI.FI team. Always refer to the oracle system's official documentation for the most accurate and up-to-date information.

### [OutputSettlerSimple](https://github.com/openintentsframework/oif-contracts/blob/main/src/output/simple/OutputSettlerSimple.sol)

OutputSettlerSimple is an [Output Settler](/lifi-intents/architecture/output-settlement) that can be used as an oracle for a same chain intent. To use the OutputSettlerSimple, call [`setAttestation`](https://github.com/openintentsframework/oif-contracts/blob/035fa5782829cb54bbd461e6e45e3f4e7d4afc8a/src/output/OutputSettlerBase.sol#L324-L354) after filling the output.

```solidity theme={"system"}
function setAttestation(
  bytes32 orderId,
  bytes32 solver,
  uint32 timestamp,
  MandateOutput calldata output
) external
```

If the Output Settler is configured as both the input and output oracle, calling `setAttestation` will expose the filled outputs for the Input Settlement to verify.

### [Polymer](https://polymerlabs.org/)

Polymer is an event based system that provides real time proofs at low cost. Security is based on bundled state through Polymer with block headers sourced from Sequencer preconfirmations.

* Low gas cost.
* Low latency for quick settlement.
* Polymer can only prove one output at a time requiring batch proving to be implemented on the solver side to optimise efficiency.
* Broadcast

The target event for validation is [`OutputFilled`](https://github.com/openintentsframework/oif-contracts/blob/222989b15241ec680d2a0503c7396e1b6c649a50/src/output/OutputSettlerBase.sol#L92-L94), emitted when an output has been filled.

```solidity theme={"system"}
event OutputFilled(
  bytes32 indexed orderId,
  bytes32 solver,
  uint32 timestamp,
  MandateOutput output,
  uint256 finalAmount
);
```

Using the Polymer [Prove API](https://docs.polymerlabs.org/docs/build/get%20started/prove-api-V2/api-endpoints), a proof of the event can be generated. Once generated, it can be submitted to [`receivedMessage`](https://github.com/openintentsframework/oif-contracts/blob/222989b15241ec680d2a0503c7396e1b6c649a50/src/integrations/oracles/polymer/PolymerOracle.sol#L74-L78).

For an example of a integration for fetching Polymer proofs, see [lintent.org/polymer](https://github.com/lifinance/lintent/blob/main/src/routes/polymer/%2Bserver.ts) or the [receiveMessage](https://github.com/lifinance/lintent/blob/5787de1e54217174062a3c8f5105bb5cdd189cf1/src/lib/utils/lifiintent/tx.ts#L668-L675) call made.

### [Wormhole](https://wormhole.com/)

Wormhole is a messaging based system that allows contracts to broadcast VAAs (Verified Action Approvals). VAAs are messages signed by the Wormhole Guardian network.

* Security through a decentralized guardian set.
* Supports batch proving of multiple outputs in a single VAA.
* Comparatively slow compared to alternatives.
* Broadcast

The Wormhole implementation is based on the broadcast functionality of Wormhole. Messages must be submitted to the Wormhole Implementation via the [`submit`](https://github.com/openintentsframework/oif-contracts/blob/222989b15241ec680d2a0503c7396e1b6c649a50/src/integrations/oracles/wormhole/WormholeOracle.sol#L48-L53) interface. Messages must be encoded into [`FillDescription`](https://github.com/openintentsframework/oif-contracts/blob/222989b15241ec680d2a0503c7396e1b6c649a50/src/libs/MandateOutputEncodingLib.sol#L22-L37)s and then submitted:

```solidity theme={"system"}
function submit(
  address source,
  bytes[] calldata payloads
) public payable returns (uint256 refund);
```

This message is then emitted to the Wormhole guardian set. Once the associated proof becomes available, the solver can submit the proof to [`receiveMessage`](https://github.com/openintentsframework/oif-contracts/blob/daa8913e5803d8b62b646335d4c5130cdfacfec8/src/oracles/wormhole/WormholeOracle.sol#L78-L80) on the input chain to validate their intents.

There are many ways to get Wormhole VAAs like the [Wormholescan api](https://docs.wormholescan.io) or the [Wormhole SPY](https://wormhole.com/docs/protocol/infrastructure/spy/).

<Note>
  The Wormhole implementation uses a significantly more efficient validation algorithm than Wormhole's `Implementation.sol` for gas saving purposes.
</Note>

### [Hyperlane](https://www.hyperlane.xyz/)

Hyperlane is a highly customizable messaging based system that allows contracts to send and receive messages verified through a selected Interchain Security Modules (ISM).

* Highly customizable security modules.
* Supports batch proving of multiple outputs in a single message.
* Strong censorship guarantees.
* Automatic or manual relaying.

The Hyperlane implementation is based on the `dispatch` function on the Hyperlane mailbox. To configure Hyperlane relaying, provide the relevant `customHook` and `customHookMetadata` when calling [`submit`](https://github.com/openintentsframework/oif-contracts/blob/222989b15241ec680d2a0503c7396e1b6c649a50/src/integrations/oracles/hyperlane/HyperlaneOracle.sol#L60-L116).

```solidity theme={"system"}
function submit(
  uint32 destinationDomain,
  address recipientOracle,
  uint256 gasLimit,
  bytes calldata customMetadata,
  address source,
  bytes[] calldata payloads
) public payable
```

For more about dispatch hooks see the [Hyperlane documentation](https://docs.hyperlane.xyz/docs/protocol/core/post-dispatch-hooks-overview).

Because Hyperlane is a highly customizable system, the exact required interactions highly varies based on the selected ISM and relaying method. For more information, refer to the [Hyperlane documentation](https://docs.hyperlane.xyz).

### [Chainlink CCIP](https://chain.link/cross-chain)

Chainlink CCIP is a messaging based system that allows contracts to send and receive messages verified through Chainlink oracles.

* Security through Chainlink's oracle network.
* Supports batch proving of multiple outputs in a single message.
* [SOC2 and ISO 27001](https://blog.chain.link/chainlink-achieves-iso-soc-2-certifications/) compliant

Chainlink CCIP implementation is based on the CCIP router `ccipSend` with an `EVM2AnyMessage` . To submit filled outputs to the CCIP oracle, first call the quote endpoint: `getFee` to estimate the messaging fee. Then call `submit` to send the message to the output chain along with the required fee. If an ERC20 token is used to pay the fee, approve the token transfer first.

```solidity theme={"system"}
function submit(
  uint64 destinationChainSelector,
  bytes32 receiver,
  bytes calldata extraArgs,
  address source,
  bytes[] calldata payloads,
  address feeToken
)
```

CCIP is automatic and once submitted, Chainlink will relay the message to the input chain.

### [Axelar](https://www.axelar.network)

Axelar is a messaging based system that allows contracts to send and receive message verified by Axelar's Proof of State network.

* Security through Axelar's Proof of State network.
* Supports batch proving of multiple outputs in a single message.
* Automatic relaying.

Axelar implementation is based on the Axelar Gateway `callContract` function. To submit filled outputs to the Axelar oracle, first use the `gasService` to estimate the fees. Then call `submit` to send the message to the output chain along with the required fee.

```solidity theme={"system"}
submit(
  string calldata destinationChain,
  string calldata destinationAddress,
  address source,
  bytes[] calldata payloads
) public payable
```

Axelar is automatic and once submitted, Axelar will relay the message to the input chain.

### [LayerZero](https://layerzero.network/)

LayerZero is a security customizable messaging based system that allows contracts to send and receive messages verified through a selected set of Decentralized Verifier Networks (DVNs).

* Customizable security
* Popular and broad chain compatibility.
* Supports batch proving of multiple outputs in a single message.
* Expensive

The LayerZero implementation is based on `send` on the LayerZero endpoint. To submit filled outputs to the LayerZero oracle, first use the `quote` function to estimate fees and then call `submit` with the required fee to send the message to the output chain.

```solidity theme={"system"}
function submit(
  uint32 dstEid,
  address recipientOracle,
  address source,
  bytes[] calldata payloads,
  bytes calldata options
) external payable;
```

LayerZero is automatic and once submitted, LayerZero will relay the message to the input chain.

### Bitcoin

LI.FI Intents has a Bitcoin Simplified Payment Validation (SPV) client implementation. This implementation works both as an Output Settlement implementation and as a validation layer.

The Bitcoin SPV client requires constant upkeep. The blockchain must be updated approximately every 10 minutes, or whenever a transaction needs to be proven, to properly validate transactions.

To generate a transaction proof, refer to the code below:

```typescript theme={"system"}
import mempoolJS from "@catalabs/mempool.js";
const mainnet: boolean;
const {
  bitcoin: { transactions, blocks },
} = mempoolJS({
  hostname: "mempool.space",
  network: mainnet ? undefined : "testnet4",
});

export async function generateProof(
  txid: string,
): Promise<{ blockHeader: string; proof: Proof; rawTx: string }> {
  const tx = await transactions.getTx({ txid });

  const merkleProof = await transactions.getTxMerkleProof({ txid });
  // TODO: serialization version 1.
  const rawTx = await transactions.getTxHex({ txid });

  const blockHash = await blocks.getBlockHeight({
    height: merkleProof.block_height,
  });

  // Most endpoints provide transactions witness encoded.
  // The following function serves to strip the witness data.
  const rawTxWitnessStripped = removeWitnesses(rawTx);

  const blockHeader = await blocks.getBlockHeader({ hash: blockHash });

  return {
    blockHeader,
    proof: {
      txId: txid,
      txIndex: merkleProof.pos,
      siblings: merkleProof.merkle.reduce(
        (accumulator, currentValue) => accumulator + currentValue,
      ),
    },
    rawTx: rawTxWitnessStripped,
  };
}
```
