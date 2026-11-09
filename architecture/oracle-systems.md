> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Oracle Systems

> LI.FI Intents lets you plug in custom validation and oracle layers, empowering you to choose the optimal trust and speed tradeoffs for your cross-chain use case.

You can plug in any validation layer that can validate a payload from the output chain. LI.FI intent supports any oracle that the Open Intents Framework (OIF) supports. An OIF oracle validation layer needs to support the following:

1. A submission interface where solvers can submit their filled outputs. It should accept arbitrary payloads for validation, then call the associated output settlement contract to check whether those payloads are valid. If the oracle can prove contract calls, events, or storage on its own, submission is automatic and this requirement does not apply.

   Oracle implementations can validate whether an output has been filled through the `IAttester` interface:

   ```solidity theme={"system"}
   interface IAttester {
     function hasAttested(
         bytes[] calldata payloads
     ) external view returns (bool);
   }
   ```

2. Implement the validation interface so the input chain can check whether an output was filled:
   ```solidity theme={"system"}
     interface IInputOracle {
         function efficientRequireProven(
           bytes calldata proofSeries
       ) external view;
     }
   ```

Beware of the following when implementing oracle systems:

* Submission and receive interfaces do not have to be standardized, but you must document them so solvers can implement them.
* Oracle systems can either have automatic relaying or manual relaying.
* The oracle system's trust assumptions must be explicit, so intent issuers and solvers can make informed decisions.

For more documentation refer to the [OIF Contributing Guidelines](https://github.com/openintentsframework/oif-contracts/blob/main/CONTRIBUTING.md#oif-oracle-pr-requirements). These requirements specify how a proper OIF compatible oracle system should behave.

## Implemented Validation Interfaces

There are three types of validation interfaces:

1. **Self-serve**: Validation interfaces where submitting the payload generates an off-chain proof that must be collected and submitted on the input chain.
   * [Polymer](#polymer) — EVM, Tron, and Solana
   * [Wormhole](#wormhole) — EVM and Solana (see the current mainnet deployment below)

2. **Automatic**: Validation interfaces where submitting the payload automatically delivers the associated proof on the input chain.
   * [Hyperlane](#hyperlane)
   * [Chainlink CCIP](#chainlink-ccip)
   * [Axelar](#axelar)
   * [LayerZero](#layerzero)

3. **Same Chain**: Using the [Output Settlement](https://docs.li.fi/lifi-intents/architecture/settlement.md#output-settlement) to produce an oracle proof for itself, skipping a dedicated oracle system.
   * [OutputSettlerSimple](#outputsettlersimple)

Choosing an oracle system is up to the intent issuer. The oracle's security is what stands behind delivery: it proves the outputs really arrived on the destination chain before the user's inputs are released to the solver. A weak oracle can let the inputs be paid out for a delivery that never happened.

Oracles reach that proof in different ways, with different tradeoffs in trust model, speed, and cost, so choose the one whose tradeoffs suit your intent. Whichever you choose, solvers have to support that oracle, or the intent will not be filled.

> [!NOTE]
>
> **Speed & Price**\
> Some oracle systems are significantly faster than other oracle systems. While speed does not impact asset delivery for users, it matters for solvers. Choosing an oracle system with fast repayments results in cheaper intents, as solvers can rotate their capital faster.

The following documentation may be outdated if the oracle network updates their documentation and does not notify the LI.FI team. Always refer to the oracle system's official documentation for the most accurate and up-to-date information.

### [OutputSettlerSimple](https://github.com/openintentsframework/oif-contracts/blob/main/src/output/simple/OutputSettlerSimple.sol)

You can use OutputSettlerSimple, an [Output Settler](https://docs.li.fi/lifi-intents/architecture/settlement.md#output-settlement), as the oracle for a same-chain intent. To use it, call [`setAttestation`](https://github.com/openintentsframework/oif-contracts/blob/035fa5782829cb54bbd461e6e45e3f4e7d4afc8a/src/output/OutputSettlerBase.sol#L324-L354) after filling the output.

```solidity theme={"system"}
function setAttestation(
  bytes32 orderId,
  bytes32 solver,
  uint32 timestamp,
  MandateOutput calldata output
) external
```

If the Output Settler is configured as both the input and output oracle, calling `setAttestation` will expose the filled outputs for the Input Settlement to verify.

#### Same-chain intents on Solana

On Solana, no `setAttestation` step exists — the fill itself creates the proof. `output_settler_simple::fill` (or `native_fill`) creates a [`LocalAttestation`](https://docs.li.fi/lifi-intents/knowledge-database/glossary#localattestation) PDA when a fill succeeds, and `finalise` validates that account directly. No oracle program is involved, and there is no callback flow. Instead, a single-output order can use [input-first settlement](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#input-first-settlement-finalise_with_prefill): the escrow pays the solver and a compact fill delivers the output in the same transaction, with no attestation at all. See [Attestations](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#attestations) for the account model.

### [Polymer](https://polymerlabs.org/)

Polymer is an event based system that provides real time proofs at low cost. Security is based on bundled state through Polymer with block headers sourced from Sequencer preconfirmations.

* Low gas cost.
* Low latency for quick settlement.
* Polymer can only prove one output at a time, so solvers have to implement batch proving themselves to stay efficient.
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

Using the Polymer [Prove API](https://docs.polymerlabs.org/docs/build/get%20started/prove-api-V2/api-endpoints), you can generate a proof of the event, then submit it to [`receiveMessage`](https://github.com/openintentsframework/oif-contracts/blob/222989b15241ec680d2a0503c7396e1b6c649a50/src/integrations/oracles/polymer/PolymerOracle.sol#L74-L78).

For an example integration that fetches Polymer proofs, see [lintent.org/polymer](https://github.com/lifinance/lintent/blob/main/src/routes/polymer/%2Bserver.ts) or the [receiveMessage](https://github.com/lifinance/lintent/blob/5787de1e54217174062a3c8f5105bb5cdd189cf1/src/lib/utils/lifiintent/tx.ts#L668-L675) call made.

#### Polymer on Solana

The Solana Polymer oracle is the `oracle_polymer` Anchor program (deployed name: `polymer`; IDL: [polymer.json](https://docs.li.fi/idl/polymer.json)). The flow matches the EVM one — validate fills, emit a provable log, verify the proof on the input chain — with Solana mechanics:

* **Output side (a fill on Solana)**: after filling, the solver exports the fill with one of two instructions. Both validate each description against its `LocalAttestation` and emit a `Prove:` log line that Polymer proves on the input chain:

  * `submit(source, fill_descriptions)` — permissionless and non-consuming. Its only named account is the `oracle_polymer` state PDA; `remaining_accounts` holds one read-only attestation per payload. The attestation stays in place, so a dropped or unproven log can simply be submitted again.
  * `submit_consume(source, fill_descriptions)` — also consumes each attestation, which closes it and returns its rent (if the recorded payer is an executable account, the attestation is only marked consumed and the rent is [deferred](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#consuming-local-attestations)). Named accounts: `submitter` (signer), `oracle_polymer`, Polymer's `local_consumer` PDA (`2m1udXnYmaxjSXBBE88yZZcwrtT127DqLcwK9B3WCAus`), and `intents_protocol`. `remaining_accounts` holds interleaved writable pairs `[attestation_0, rent_refund_0, attestation_1, rent_refund_1, …]`. For every fill payload the `submitter` must be the attestation's recorded rent payer — the Solana account that signed the fill, not the `filler_data` solver id — otherwise it fails with `UnauthorizedFillSubmitter`. A consuming batch therefore cannot mix fills from different fillers. Not-filled payloads can be submitted by anyone.

  A consumed attestation is no longer a valid proof, so it cannot be exported again by either instruction. Consume only once you no longer need to resubmit — for example, after a `submit` whose log you have already proven.
* **Input side (a Solana order awaiting a remote proof)**: receiving always ends in `receive_attest()`, which reads the validated event from the prover's result account, maps the source chain id through `ChainMapping`, and registers the [`Attestation`](https://docs.li.fi/lifi-intents/knowledge-database/glossary#attestation).

For a Solana **output**, the proof pipeline is:

**1. Fill, then submit on Solana**

`fill` creates the `LocalAttestation`; `oracle_polymer::submit` (or `submit_consume`) validates it and emits the `Prove:` log. They can be separate transactions or combined into one if it fits; what matters is which transaction carries the `Prove:` log.

**2. Request the proof of the submit transaction**

Ask Polymer's Prove API to prove the **transaction that contains the `Prove:` log** — the one with `submit`, not a fill-only transaction. Requesting a proof of a transaction without the log fails with "transaction does not contain a Polymer Prove log".

Set `srcChainId` to `2`. This is Polymer's own protocol id for Solana, not the canonical LI.FI chain id — see [chain id namespaces](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md#chain-ids).

**3. Deliver to receiveSolanaMessage on the input chain**

Solana-sourced proofs go to the EVM oracle's `receiveSolanaMessage`, **not** `receiveMessage`. The two entry points parse different payload formats and write to different storage, so a proof delivered to the wrong one reverts. Select the entry point based on the output chain type.

For a Solana **input** receiving a remote proof, real Polymer proofs are typically several kilobytes — larger than a Solana transaction — so the common path loads the proof in chunks:

1. Call `create_polymer_accounts()`.
2. Call `load_proof_chunk(chunk)` once per chunk.
3. Call `validate_event()` with a compute budget of roughly 1,000,000 CU.
4. Call `receive_attest()`.

For an unchunked proof that fits in one transaction, use two transactions instead: `receive_load_proof(proof)`, then `receive_attest()`. In both paths, `receive_attest()` closes the prover's scratch PDAs and returns their rent to the signer, who funded it.

> [!NOTE]
>
> The prover's scratch accounts are derived from the signer alone, so two concurrent receive flows under one signer collide. The second `create_polymer_accounts` fails while the accounts exist, and interleaved chunk loads write into the same proof buffer and corrupt it. Serialize receives per signer, or use distinct signers for parallel proofs.

> [!WARNING]
>
> **Identity trap**: on remote chains, the Polymer oracle is identified by its **program id** (Polymer proves the `returnedProgramId` of the log-emitting program), while the Wormhole oracle is identified by its **oracle PDA** (the VAA emitter). Use the right identifier when setting `MandateOutput.oracle` and configuring remote-side oracles — the full per-field rules are in [Which identity goes in which field](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#which-identity-goes-in-which-field).

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

> [!NOTE]
>
> The Wormhole implementation uses a significantly more efficient validation algorithm than Wormhole's `Implementation.sol`, to save gas.

#### Deployments

The Wormhole oracle is deployed on EVM mainnets; see the [Wormhole Oracle Mainnet address table](https://docs.li.fi/lifi-intents/architecture/overview.md#wormhole-oracle-mainnet). The Solana implementation is deployed on mainnet; see the [Solana program address](https://docs.li.fi/lifi-intents/architecture/overview.md#solana).

* On EVM, `submit` publishes the Wormhole message with consistency level `15`.
* Fetch the signed VAA from the [Wormholescan API](https://docs.wormholescan.io).
* Delivery is not automatic: the solver (or any caller) must submit the VAA to `receiveMessage` on the input chain.

#### Wormhole on Solana

The Solana Wormhole oracle implementation is the `oracle_wormhole` Anchor program (program name: `wormhole`; IDL: [wormhole.json](https://docs.li.fi/idl/wormhole.json)). Its mainnet deployment is listed in the [deployment overview](https://docs.li.fi/lifi-intents/architecture/overview.md#solana):

* **Output side**: `submit(source, fill_descriptions)` validates **and consumes** the fills' `LocalAttestation` accounts, then posts one Wormhole message covering all of them (finality: finalized). `remaining_accounts` holds interleaved writable pairs `[attestation_0, rent_refund_0, …]`, and the batch must not be empty. The named `submitter` and a fresh `post_message` account sign; there is no filler-signature requirement. The VAA emitter is the oracle's PDA.
* **Input side**: the `receive` instruction verifies a posted VAA, maps the emitter chain via `ChainMapping`, and creates one `Attestation` per payload. `remaining_accounts` holds one writable destination attestation PDA per decoded payload, in payload order, with no rent-refund slots; the named signer funds their creation.

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

Because Hyperlane is so customizable, what you actually have to do varies a lot with the ISM and relaying method you choose. For more information, refer to the [Hyperlane documentation](https://docs.hyperlane.xyz).

### [Chainlink CCIP](https://chain.link/cross-chain)

Chainlink CCIP is a messaging based system that allows contracts to send and receive messages verified through Chainlink oracles.

* Security through Chainlink's oracle network.
* Supports batch proving of multiple outputs in a single message.
* [SOC2 and ISO 27001](https://blog.chain.link/chainlink-achieves-iso-soc-2-certifications/) compliant

The Chainlink CCIP implementation sends an `EVM2AnyMessage`, with no token transfers, through `ccipSend` on the CCIP router. To submit filled outputs, first call [`getFee`](https://github.com/catalystsystem/lifi-intent/blob/1f8ec4ee3df0e4c1f50e51cc737845eb279babc1/src/integrations/oracles/chainlink/ChainlinkCCIPOracle.sol) on the output chain's oracle to quote the messaging fee. Then call `submit` with the same arguments and the fee to send the message to the input chain.

```solidity theme={"system"}
function getFee(
  uint64 destinationChainSelector,
  bytes32 recipientOracle,
  bytes calldata extraArgs,
  address feeToken,
  address source,
  bytes[] calldata payloads
) external view returns (uint256 fee);

function submit(
  uint64 destinationChainSelector,
  bytes32 recipientOracle,
  bytes calldata extraArgs,
  address feeToken,
  address source,
  bytes[] calldata payloads
) external payable returns (uint256 refund);
```

* `destinationChainSelector` is the input chain's CCIP chain selector, not its chain id. See the [address table](https://docs.li.fi/lifi-intents/architecture/overview.md#chainlink-ccip-oracle-mainnet) or the [CCIP directory](https://docs.chain.link/ccip/directory/mainnet).
* `recipientOracle` is the input chain's CCIP oracle address, left-padded to 32 bytes.
* `extraArgs` are the CCIP extra args, such as the destination gas limit. Empty bytes use the CCIP defaults.
* `feeToken` set to `address(0)` pays the fee in the native token from `msg.value`. With an ERC20 fee token, approve the oracle for the fee first; the oracle pulls the fee from the caller. In both cases, any `msg.value` not spent on the fee is refunded to the caller.

CCIP is automatic: once submitted, Chainlink relays the message to the input chain, where the oracle records the outputs as proven.

#### Deployments

The Chainlink CCIP oracle is deployed on EVM mainnets; see the [Chainlink CCIP Oracle Mainnet address table](https://docs.li.fi/lifi-intents/architecture/overview.md#chainlink-ccip-oracle-mainnet).

* On the input chain, the oracle identifies the remote oracle by the 32-byte CCIP sender, the output chain's oracle address left-padded.
* Each oracle maps CCIP chain selectors to chain ids. A message from a source chain whose selector is not mapped reverts on delivery. It can be executed manually through CCIP once the selector is mapped.

### [Axelar](https://www.axelar.network)

Axelar is a messaging based system that allows contracts to send and receive message verified by Axelar's Proof of State network.

* Security through Axelar's Proof of State network.
* Supports batch proving of multiple outputs in a single message.
* Automatic relaying.

The Axelar implementation sends the message through `callContract` on the Axelar Gateway. To submit filled outputs, call [`submit`](https://github.com/catalystsystem/lifi-intent/blob/1f8ec4ee3df0e4c1f50e51cc737845eb279babc1/src/integrations/oracles/axelar/AxelarOracle.sol) on the output chain's oracle.

```solidity theme={"system"}
enum DeliveryMode { Relayed, SelfRelay }

function submit(
  string calldata destinationChain,
  string calldata recipientOracle,
  address source,
  bytes[] calldata payloads,
  bytes32 destinationConfig,
  DeliveryMode deliveryMode
) external payable
```

* `destinationChain` is the input chain's Axelar chain name, for example `Ethereum`, `binance` or `hyperliquid`. See the [address table](https://docs.li.fi/lifi-intents/architecture/overview.md#axelar-oracle-mainnet) or [Axelar's chain config](https://github.com/axelarnetwork/axelar-contract-deployments/blob/8528e73bfd7cb04b55649a7d8e693c314d5dbb4c/axelar-chains-config/info/mainnet.json). The oracle lowercases the name only to look up its chain map and forwards your spelling to Axelar unchanged, so pass the name exactly as Axelar lists it.
* `recipientOracle` is the input chain's Axelar oracle address as a `0x` hex string. It is passed to Axelar unchecked.
* `destinationConfig` must be `bytes32(0)` for EVM destinations. A nonzero value wraps the message in the Solana envelope.
* `deliveryMode` selects how the message is delivered:
  * `Relayed` pays `msg.value` to the Axelar Gas Service, and Axelar's relayer executes the message on the input chain. `msg.value` must be nonzero; estimate it with Axelar's [gas estimation](https://docs.axelar.dev/dev/gas-service/intro). Unused gas is refunded to the caller.
  * `SelfRelay` sends no value (`msg.value` must be zero). Once Axelar approves the message, the solver (or anyone) must call `execute` on the input chain's oracle.

```solidity theme={"system"}
function execute(
  bytes32 commandId,
  string calldata sourceChain,
  string calldata sourceAddress,
  bytes calldata payload
) external
```

`execute` checks the message with the Axelar Gateway, maps `sourceChain` through the oracle's chain map, and records the outputs as proven. The remote oracle is identified by `sourceAddress`.

`submit` consumes nothing, so a delivery lost to a wrong chain name, recipient or too little gas can be submitted again.

#### Deployments

The Axelar oracle is deployed on EVM mainnets; see the [Axelar Oracle Mainnet address table](https://docs.li.fi/lifi-intents/architecture/overview.md#axelar-oracle-mainnet).

* Each oracle maps lowercase Axelar chain names to chain ids. A message to or from a chain that is not mapped reverts.
* With `Relayed` delivery, Axelar is automatic: once submitted, Axelar relays the message to the input chain.

### [LayerZero](https://layerzero.network/)

LayerZero is a security customizable messaging based system that allows contracts to send and receive messages verified through a selected set of Decentralized Verifier Networks (DVNs).

* Customizable security
* Popular and broad chain compatibility.
* Supports batch proving of multiple outputs in a single message.
* Expensive

The LayerZero implementation is based on `send` on the LayerZero endpoint. To submit filled outputs to the LayerZero oracle, first use the `quote` function to estimate fees, then call `submit` on the output chain, with the required fee, to send the message to the input chain.

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

LI.FI Intents has a Bitcoin Simplified Payment Validation (SPV) client implementation. It works both as an output settlement and as a validation layer.

The Bitcoin SPV client requires constant upkeep. To validate transactions properly, update the chain roughly every 10 minutes, and whenever you need to prove a transaction.

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


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.