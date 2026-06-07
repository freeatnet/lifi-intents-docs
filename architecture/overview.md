> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# System Overview

> Overview of the LI.FI Intent System. The system consists of three modular components (Input Settler, Output Settler, and validation network) that can be selected based on the desired intent structure.

The LI.FI Intent System modularizes three components of the intent flow:

1. **Input Settler** – Handles the source chain funds.
2. **Output Settler** – Handles the destination chain funds.
3. **Oracle System** – Makes Output Settlement statements available for the input settler.

<img src="https://mintcdn.com/lifi/pOTDBSGpduSCqu2-/images/lifi-intent-big.png?fit=max&auto=format&n=pOTDBSGpduSCqu2-&q=85&s=9600e4af302a6e2dbf4942035adc65d0" alt="LI.FI Intent Calls Diagram" width="2944" height="2816" data-path="images/lifi-intent-big.png" />

Historically, these components have been intertwined, which presented scaling challenges. A componentized approach allows for greater flexibility and extensibility, enabling different input and output settlement schemes and validation layers to be mixed and matched.

### Input settlement

The input settlement scheme manages user deposits and releases funds to solvers once intents are fulfilled. LI.FI Intents currently support two input settlement schemes:

* **Escrow** via [`InputSettlerEscrow.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/input/escrow/InputSettlerEscrow.sol)
* [**The Compact**](https://github.com/Uniswap/the-compact) via [`InputSettlerCompact.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/input/compact/InputSettlerCompact.sol). The Compact supports both fill-first flows and ordinary flows.

The intent system imposes no restrictions on the implementation of input settlements. Input settlements can access proven outputs through validation layers by calling [`efficientRequireProven`](https://github.com/openintentsframework/oif-contracts/blob/main/src/interfaces/IOracle.sol#L19-L27). If an order contains multiple outputs and the outputs are filled by different solvers, **the filler of the first output** in the order specification is considered the canonical solver.

[Learn more about Input Settlement →](/lifi-intents/architecture/input-settlement)

### Output settlement

The output settlement scheme handles the delivery of assets on destination chains. It imposes no interface requirements, order structure, or order type, except that an [interface to validate payloads](https://github.com/openintentsframework/oif-contracts/blob/main/src/interfaces/IPayloadCreator.sol) must be provided:

```solidity theme={"system"}
interface IPayloadCreator {
    function arePayloadsValid(
        bytes[] calldata payloads
    ) external view returns (bool);
}
```

This allows the output settlement scheme to be incredibly flexible; it can support any order type on any virtual machine, as long as the filled order can be expressed as an opaque bytes array.

The initial version of the intent system uses `MandateOutput`, with the encoding described by [`MandateOutputEncodingLib.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/libs/MandateOutputEncodingLib.sol).

If the input settlement could validate this call, the inputs could be appropriately paid to the solver. However, this information only exists on the output settlement on the output chain.

[Learn more about Output Settlement →](/lifi-intents/architecture/output-settlement)

### Oracle system

The oracle system ferries valid payloads from the output chain to the input chain. It serves as the bridge that confirms output delivery has occurred.

Any validation layer can be added as long as it supports validating a payload from a remote chain. Currently available validation layers include:

* [**Polymer**](https://github.com/openintentsframework/oif-contracts/blob/main/src/oracles/polymer/PolymerOracleMapped.sol)
* [**Wormhole**](https://github.com/openintentsframework/oif-contracts/blob/main/src/oracles/wormhole/WormholeOracle.sol)

Before emitting messages, the oracle is expected to check if one or more payloads are valid and then ship them to the input chain:

```solidity theme={"system"}
function submit(address proofSource, bytes[] calldata payloads) external payable {
    // Check if the payloads are valid.
    if (!IPayloadCreator(proofSource).arePayloadsValid(payloads)) revert NotAllPayloadsValid();

    // Payloads are valid. We can submit them on behalf of proofSource.
    _submit(proofSource, payloads);
}
```

Oracles may use custom message encodings, custom relaying properties, custom interfaces, or other special integration concerns. Internal oracle messaging is not standardized.

On the input chain, the oracle system is expected to validate the payload(s) through a virtual machine local hash:

```solidity theme={"system"}
interface IValidationLayer {
    /**
     * @notice Check if a series of data has been attested to.
     * @dev Does not return a boolean; instead, it reverts if false.
     * This function returns true if proofSeries is empty.
     * @param proofSeries remoteOracle, remoteChainId, and dataHash encoded in chunks of 32*4=128 bytes.
     */
    function efficientRequireProven(
        bytes calldata proofSeries
    ) external view;
}
```

Using only the payload hash makes the system more efficient, as less data is passed around. No attempt is made at standardizing the payload. As a result, there may be [incompatibilities](/lifi-intents/architecture/oracle-systems) between the input and output settlement layers.

[Learn more about Validation Layers →](/lifi-intents/architecture/oracle-systems)

## Security Assumptions

The intent system includes resource locks, which create trust boundaries between key actors:

* **[Sponsors](/lifi-intents/knowledge-database/glossary#sponsor)** (users) trust that arbiters won't fraudulently finalize issued locks.
* **[Arbiters](/lifi-intents/knowledge-database/glossary#arbiter)** and **[solvers](/lifi-intents/knowledge-database/glossary#solver)** trust allocators not to co-sign overlapping locks exceeding user deposits.

No single actor can independently access funds, creating a secure environment for cross-chain transactions.

For more about security and intent validation, [see order-validation →](/lifi-intents/for-solvers/orderflow#order-validation)

## Integration Points

LI.FI intent is designed to be highly composable, with different components capable of being swapped out as needed:

* **Input Settlement**: New resource lock standards and standard intent interfaces can be integrated.
* **Output Settlement**: New order types and fulfillment mechanisms can be added.
* **Oracle System**: Different cross-chain messaging protocols and proof layers can be supported.

[Integrators](/lifi-intents/knowledge-database/glossary#integrator) can freely pick and choose which components to use for which swaps. This gives intent issuers maximum flexibility to describe their desired end state.

### Same-chain intents

LI.FI Intents were designed for composable cross-chain swaps and also support same-chain intents in a compatible envelope. In same-chain mode, the output settler can act as the oracle path.

To use LI.FI Intents for same-chain swaps, configure the output settler as both input and output oracle. Solvers filling same-chain intents can optionally use a callback to receive inputs before delivering outputs.

<Columns cols={2}>
  <Frame caption="Same Chain Intent">
    <img src="https://mintcdn.com/lifi/UpfTKvFmL7B7fUrN/images/lifi-intent-same-chain.png?fit=max&auto=format&n=UpfTKvFmL7B7fUrN&q=85&s=ad15cd965b5b527e75c248c8c8fda38f" alt="Same Chain Intent without Callback" width="1856" height="1984" data-path="images/lifi-intent-same-chain.png" />
  </Frame>

  <Frame caption="Same Chain Intent with callback">
    <img src="https://mintcdn.com/lifi/UpfTKvFmL7B7fUrN/images/lifi-intent-same-chain-callback.png?fit=max&auto=format&n=UpfTKvFmL7B7fUrN&q=85&s=ffab84b6b99a28227cade279707aaecd" alt="Same Chain intent with Callback" width="1914" height="1984" data-path="images/lifi-intent-same-chain-callback.png" />
  </Frame>
</Columns>

To receive inputs before delivering outputs, your solver has to support `orderFinalised`. When using the callback to receive inputs, it is important to `fill` and `setAttestations` for the intent outputs within the callback.

```solidity theme={"system"}
function orderFinalised(
    uint256[2][] calldata inputs,
    bytes calldata call
) external
```

For more on same-chain callbacks, see the [same-chain test case](https://github.com/catalystsystem/catalyst-intent/blob/main/test/integration/InputSettler7683LIFI.samechain.t.sol).

For more about same chain intents, see [Output Settler as Oracle](/lifi-intents/architecture/oracle-systems#outputsettlersimple)

### Smart contracts

Currently maintained LI.FI Intent deployments are listed below. Contracts are deployed to the same addresses across supported chains and can be permissionlessly deployed to new chains.

**Mainnet (including):** Ethereum, Base, Arbitrum

**Testnet (including):** Sepolia, Base Sepolia, Arbitrum Sepolia, Optimism Sepolia

* **Input Settler Compact**: `0x0000000000cd5f7fDEc90a03a31F79E5Fbc6A9Cf`

* **Input Settler Escrow**: `0x000025c3226C00B2Cdc200005a1600509f4e00C0`

* **Output Settler**: `0x0000000000eC36B683C2E6AC89e9A75989C22a2e`

* **Polymer Oracle Mainnet**: `0x0000003E06000007A224AeE90052fA6bb46d43C9`

* **Polymer Oracle Testnet**: `0xC401b53377b8A71A7cEB820e6a4dC53832343a90`

* **Uniswap The Compact**: `0x00000000000000171ede64904551eeDF3C6C9788`
