> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# System Overview

> Overview of the LI.FI Intent System. The system consists of three modular components (Input Settler, Output Settler, and validation network) that can be selected based on the desired intent structure.

The LI.FI Intent System splits the intent flow into three components:

1. **Input Settler** – Handles the source chain funds.
2. **Output Settler** – Handles the destination chain funds.
3. **Oracle System** – Carries the output settler's statements back to the input settler.

![LI.FI Intent Calls Diagram](https://mintcdn.com/lifi/pOTDBSGpduSCqu2-/images/lifi-intent-big.png?fit=max\&auto=format\&n=pOTDBSGpduSCqu2-\&q=85\&s=9600e4af302a6e2dbf4942035adc65d0)

These components have usually been welded together, which made such systems hard to scale. Keeping them separate is more flexible: you can mix and match input settlement schemes, output settlement schemes, and validation layers.

The same three-component model runs beyond the EVM: on Tron with the same contracts, and on Solana as a set of Anchor programs — see [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md) for that implementation.

### Input settlement

The input settlement scheme manages user deposits and releases funds to solvers once intents are fulfilled. LI.FI Intents currently support two input settlement schemes:

* **Escrow** via [`InputSettlerEscrow.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/input/escrow/InputSettlerEscrow.sol)
* [**The Compact**](https://github.com/Uniswap/the-compact) via [`InputSettlerCompact.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/input/compact/InputSettlerCompact.sol). The Compact supports both fill-first flows and ordinary flows.

The intent system places no limits on how you implement input settlement. Input settlements can access proven outputs through validation layers by calling [`efficientRequireProven`](https://github.com/openintentsframework/oif-contracts/blob/03dee826c0c46d1701213d2cb2e52a019f09d423/src/interfaces/IOracle.sol#L19-L27).

If an order contains multiple outputs and different solvers fill them, **the filler of the first output** in the order specification is the canonical solver — the one who claims the order's inputs.

[Learn more about Input Settlement →](https://docs.li.fi/lifi-intents/architecture/settlement.md#input-settlement)

### Output settlement

The output settlement scheme handles the delivery of assets on destination chains. It imposes no interface requirements, order structure, or order type, except that an [interface to validate payloads](https://github.com/openintentsframework/oif-contracts/blob/3d1399025b9daa8ab6f0666f11b741d3b5c5fafa/src/interfaces/IPayloadCreator.sol) must be provided:

```solidity theme={"system"}
interface IPayloadCreator {
    function arePayloadsValid(
        bytes[] calldata payloads
    ) external view returns (bool);
}
```

This allows the output settlement scheme to be incredibly flexible; it can support any order type on any virtual machine, as long as the filled order can be expressed as an opaque bytes array.

The initial version of the intent system uses `MandateOutput`, with the encoding described by [`MandateOutputEncodingLib.sol`](https://github.com/openintentsframework/oif-contracts/blob/main/src/libs/MandateOutputEncodingLib.sol).

The input settlement could pay the solver if it could validate this call — but the information it needs only exists on the output settlement, on the output chain. Closing that gap is what the oracle system does.

[Learn more about Output Settlement →](https://docs.li.fi/lifi-intents/architecture/settlement.md#output-settlement)

### Oracle system

The oracle system ferries valid payloads from the output chain to the input chain. It serves as the bridge that confirms output delivery has occurred.

Any validation layer can be added as long as it supports validating a payload from a remote chain. Currently available validation layers include:

* [**Polymer**](https://github.com/openintentsframework/oif-contracts/blob/main/src/integrations/oracles/polymer/PolymerOracleMapped.sol)
* [**Wormhole**](https://github.com/openintentsframework/oif-contracts/blob/main/src/integrations/oracles/wormhole/WormholeOracle.sol)
* [**Bitcoin**](https://github.com/openintentsframework/oif-contracts/blob/main/src/integrations/oracles/bitcoin/BitcoinOracle.sol)

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

On the input chain, the oracle system is expected to validate the payloads through a hash that is local to that virtual machine:

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

Using only the payload hash makes the system more efficient, since less data moves around. The payload itself is not standardized. As a result, there may be [incompatibilities](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md) between the input and output settlement layers.

[Learn more about Validation Layers →](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md)

## Security Assumptions

The intent system includes resource locks, which create trust boundaries between key actors:

* **[Sponsors](https://docs.li.fi/lifi-intents/knowledge-database/glossary.md#sponsor)** (users) trust that arbiters won't fraudulently finalize issued locks.
* **[Arbiters](https://docs.li.fi/lifi-intents/knowledge-database/glossary.md#arbiter)** and **[solvers](https://docs.li.fi/lifi-intents/knowledge-database/glossary.md#solver)** trust allocators not to co-sign overlapping locks exceeding user deposits.

No single actor can independently access funds, creating a secure environment for cross-chain transactions.

For more about security and intent validation, [see order-validation →](https://docs.li.fi/lifi-intents/for-solvers/orderflow.md#order-validation)

## Integration Points

LI.FI intents are highly composable — you can swap out any of the components:

* **Input Settlement**: New resource lock standards and standard intent interfaces can be integrated.
* **Output Settlement**: New order types and fulfillment mechanisms can be added.
* **Oracle System**: Different cross-chain messaging protocols and proof layers can be supported.

[Integrators](https://docs.li.fi/lifi-intents/knowledge-database/glossary.md#integrator) can freely pick and choose which components to use for which swaps. This gives intent issuers maximum flexibility to describe their desired end state.

### Same-chain intents

LI.FI Intents were designed for composable cross-chain swaps, and they also support same-chain intents in the same order format. For a same-chain intent, the output settler can act as the oracle.

To use LI.FI Intents for same-chain swaps, configure the output settler as both input and output oracle. Solvers filling same-chain intents can optionally use a callback to receive inputs before delivering outputs.

![Same Chain Intent without Callback](https://mintcdn.com/lifi/UpfTKvFmL7B7fUrN/images/lifi-intent-same-chain.png?fit=max\&auto=format\&n=UpfTKvFmL7B7fUrN\&q=85\&s=ad15cd965b5b527e75c248c8c8fda38f)

Same Chain Intent

![Same Chain intent with Callback](https://mintcdn.com/lifi/UpfTKvFmL7B7fUrN/images/lifi-intent-same-chain-callback.png?fit=max\&auto=format\&n=UpfTKvFmL7B7fUrN\&q=85\&s=ffab84b6b99a28227cade279707aaecd)

Same Chain Intent with callback

To receive inputs before delivering outputs, your solver has to support `orderFinalised`. When using the callback to receive inputs, it is important to `fill` and `setAttestations` for the intent outputs within the callback.

```solidity theme={"system"}
function orderFinalised(
    uint256[2][] calldata inputs,
    bytes calldata call
) external
```

For more on same-chain callbacks, see the [same-chain test case](https://github.com/catalystsystem/catalyst-intent/blob/main/test/integration/InputSettler7683LIFI.samechain.t.sol).

For more about same chain intents, see [Output Settler as Oracle](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md#outputsettlersimple)

### Smart contracts

Currently maintained LI.FI Intent deployments are listed below. On EVM chains, contracts are deployed to the same addresses (via a keyless CREATE2 factory) and can be permissionlessly deployed to new EVM chains.

#### EVM

* **Input Settler Escrow**: `0x00fC00edbe7C003b006f870068c548940000223e`
* **Output Settler**: `0x75220B7600c300005038432a0000f308e0000068`
* **Polymer Oracle Mainnet**: `0x008C3800F3Ad9b3B662d002E90Cc00000000eE17`
* **Polymer Oracle Testnet**: `0xC401b53377b8A71A7cEB820e6a4dC53832343a90`

##### Wormhole Oracle Mainnet

Wormhole oracle addresses differ per chain because each chain has a different Wormhole core contract.

| Chain | Address |
| - | - |
| Ethereum | `0x940a87f18Ceb24e58c447851cbB3A57C6FA8d52c` |
| BNB Chain | `0x940a87f18Ceb24e58c447851cbB3A57C6FA8d52c` |
| Polygon | `0x6941D83e74d3DA1c056E34eeD97d1FC580ED7F5C` |
| Avalanche | `0x04B92cc1d51e80621aBa749aB526A47B6Edff84A` |
| Celo | `0xF6B0113BFE73E9CDA09057C1cbCE53d40Db2273b` |
| Arbitrum | `0xe1718d27c1aBdaa009Aa8F4d3BeF3AFcE7D81f80` |
| Optimism | `0x0068F705F4b6739671021C60104239Cf94B2C75E` |
| Base | `0x935a2dd6ae8153F2c71fa8c1942e1fe0b5A82E72` |
| Tempo | `0x935a2dd6ae8153F2c71fa8c1942e1fe0b5A82E72` |
| Unichain | `0x4dA5E848E86cBa97256483507F8359De2DE33fFC` |
| Ink | `0x4dA5E848E86cBa97256483507F8359De2DE33fFC` |
| World Chain | `0xD1d0a0378A6c20280b81CE8B5C29657A4C95ce3A` |
| HyperEVM | `0x3e81FDc6AFD40A08D867C7D74CE31876F0dDE590` |
| Monad | `0x719eF3abf326AC84FFD653311B0376cfb28faeF1` |
| Arc | `0x4362aCd4bc071E99eD5E508aE1d0250553fACEA5` |
| Robinhood Chain | `0x00ab70C0A23C3959B9E6c804776C6c27C6A9D7b6` |
| Elysium | `0x9c1634E0DB7d24250A9988A2C4b1E5a3A42D429E` |

##### Chainlink CCIP Oracle Mainnet

Chainlink CCIP oracle addresses differ per chain because each chain has a different CCIP router. CCIP identifies chains by chain selector, not chain id; pass the input chain's selector to `submit`.

| Chain | CCIP chain selector | Address |
| - | - | - |
| Ethereum | `5009297550715157269` | `0x69284118109405bE0b3349F64B63Fe593BAe2F90` |
| BNB Chain | `11344663589394136015` | `0x68aE970D760A91D8759AE5bffd308E024dbF996c` |
| Polygon | `4051577828743386545` | `0x7A12d5946Cd2E1a9CB85465318e096Bf9E73423a` |
| Avalanche | `6433500567565415381` | `0x8c86B71C87a3061814e696D862e40d3e0415eD17` |
| Celo | `1346049177634351622` | `0x4C3b8ca6C52249dF1B5192A3e69e60a196ed83E6` |
| Arbitrum | `4949039107694359620` | `0x059259556522d7FA217b43460Fa647792E45F346` |
| Optimism | `3734403246176062136` | `0x0F9660314E64F6c7C51bd0Eb07B0af4ed6b73D82` |
| Base | `15971525489660198786` | `0x40d5F48d0Cc797a1e7903f394511E74A4202b5bF` |
| Tempo | `7281642695469137430` | `0x1834Ed847990a0bFe70dFe378a39dc17c8E1b515` |
| Unichain | `1923510103922296319` | `0xE6b1852543353EbB0f04DdC96922dC291b9f0c0A` |
| Ink | `3461204551265785888` | `0x9b539fBD724d4917909d54E356428c9bb8ad16Db` |
| World Chain | `2049429975587534727` | `0x934f53154223aA2D8aDa231f24a5924Bef320866` |
| HyperEVM | `2442541497099098535` | `0xE2143CD978F7cB7BC72745b2C8D200266e1f7930` |
| Monad | `8481857512324358265` | `0x28b2cEDC8EDE85B8b70A6D835240ccC6ad617D0e` |
| Arc | `6370580034781731079` | `0x0223A7Ee4c476E4BCFA24f94f3484277B50391e6` |
| Robinhood Chain | `6180753054346818345` | `0xDc32E4ab1E9B9a1FCfB11E10354b1EA64Be5e018` |
| Gnosis | `465200170687744372` | `0x11911a6Dd83D4817FA330df9cBD0823Eae97475a` |
| Mantle | `1556008542357238666` | `0x12E86Ca23c82eaEf469783634476BC49BbF21865` |
| Plasma | `9335212494177455608` | `0x51585EDE377ae672815d51CfBa6364F8ed6c9C65` |
| Katana | `2459028469735686113` | `0x6797738853Fc65034F909C6aDbe04E6D296c537C` |
| Pharos | `7801139999541420232` | `0x2c70C291d923b087C80C56Dc9C1AfC089A824A5B` |
| Stable | `16978377838628290997` | `0x55622288d5b892294C6952C0e44325C2e70AB556` |

##### Axelar Oracle Mainnet

Axelar oracle addresses depend on each chain's Axelar Gateway and Gas Service, so most chains share one address. Pass the input chain's Axelar chain name, exactly as listed, to `submit`.

| Chain | Axelar chain name | Address |
| - | - | - |
| Ethereum | `Ethereum` | `0x5453E82eA2757E4769120950e744b8cB6eC6C5f5` |
| BNB Chain | `binance` | `0xD354da3fd6374CbE9e7BBf32831Cf6ed878684e1` |
| Polygon | `Polygon` | `0x2a4AAfb8df518858C11e5bb847Bd53eA0B3b4bCF` |
| Avalanche | `Avalanche` | `0x24cb7d5B83135BE3BF3a12946e66194834C63e80` |
| Celo | `celo` | `0x23b8a6D877543b54A911bC1EA056B0dF71f760C7` |
| Arbitrum | `arbitrum` | `0x23b8a6D877543b54A911bC1EA056B0dF71f760C7` |
| Optimism | `optimism` | `0x23b8a6D877543b54A911bC1EA056B0dF71f760C7` |
| Base | `base` | `0x23b8a6D877543b54A911bC1EA056B0dF71f760C7` |
| HyperEVM | `hyperliquid` | `0x23b8a6D877543b54A911bC1EA056B0dF71f760C7` |
| Monad | `monad` | `0x23b8a6D877543b54A911bC1EA056B0dF71f760C7` |
| Arc | `arc` | `0x624A4Db725C42b33F3c77C6c5e95B0C3111389DF` |
| Mantle | `mantle` | `0x23b8a6D877543b54A911bC1EA056B0dF71f760C7` |

#### Tron

* **Input Settler Escrow**: `TXmVLCXzrhzmeCfchDPTmFF6Qe7rg3H7Kk`
* **Output Settler**: `THWDD3umarircbqo8jXxVazbpJnE25VjhN`
* **Polymer Oracle**: `TCeNWukZUoTSrgWZEMpn9X8C5NtV8Rsy6c`

#### Solana

* **Wormhole Oracle Mainnet**: `LiFixrxyzmt1Degd42wQCsQNhDNLjbrRwtwopX5VUrt`
* **Input Settler Escrow program**: `LiFiRp8RM7nJUZyUYC9FPPpDr7sAy5XPfBN6ABzBgT7`
* **Output Settler program**: `LiFiEDFjz5x1jJe9gSXNDHQW4dWt4yLXdp2VN4EiQUt`
* **Polymer Oracle program**: `LiFiBtfyPT1DnTHTAeZ2rwr5RgMrThwA5kt7KGT5nBV`
* **Intents Protocol program**: `LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK`

The four non-Wormhole program IDs above are deployed on both devnet and mainnet-beta. Solana orders and instructions also use program-derived accounts; see [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#program-ids-and-derived-accounts) for the operational addresses.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.