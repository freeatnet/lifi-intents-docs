> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Filling Orders

> Deliver output assets on EVM chains and Bitcoin to fulfill LI.FI intents.

To finalize an intent and claim the locked input funds, the solver must deliver all outputs described in the order. This page covers both EVM and Bitcoin filling.

***

## EVM Orders

Submit the intent outputs to the Output Settler specified in `output.settler` by calling `fillOrderOutputs`.

```solidity theme={"system"}
function fillOrderOutputs(
    uint32 fillDeadline,
    bytes32 orderId,
    MandateOutput[] calldata outputs,
    bytes32 proposedSolver
) external;
```

The transaction reverts if:

1. Approvals are not set
2. Caller balances are not sufficient
3. `fillDeadline` has expired
4. The first output in `outputs` has already been filled
5. Any external call in `outputs` reverts
6. The order context cannot be decoded
7. The destination chain is wrong
8. The Output Settler contract is wrong

<Note>
  An `orderId` mismatch will not cause a revert.
</Note>

After all outputs are delivered, proceed to [settlement](/lifi-intents/for-solvers/settlement) to finalize the order and receive the locked inputs.

***

## Bitcoin Orders

<Note>
  BTC support on LI.FI Intents is currently not live and still in development. If you are interested in filling BTC orders, reach out and we will assist you with onboarding.
</Note>

To determine whether an order involves a Bitcoin transaction, check `orderDto.order.outputs[].token`.

* The first 30 bytes should be `0x000000000000000000000000BC0000000000000000000000000000000000` (the 13th byte is `0xBC`)
* The 31st byte indicates the number of confirmations required before on-chain verification (`0x00` and `0x01` both mean 1 confirmation, `0x02` means 2, etc.)
* The 32nd byte is an address version identifier decoded as `uint8`

### Address Versions

| Version | Name    | Encoding            | Prefix | Hash Length |
| ------- | ------- | ------------------- | ------ | ----------- |
| 0       | Unknown | Ignore              |        |             |
| 1       | P2PKH   | Base58Check(00+PKH) | 1      | 20          |
| 2       | P2SH    | Base58Check(05+SH)  | 3      | 20          |
| 3       | P2WPKH  | Bech32              | bc1q   | 20          |
| 4       | P2WSH   | Bech32              | bc1q   | 32          |
| 5       | P2TR    | Bech32m             | bc1p   | 32          |

The `outputs[].recipient` field contains the destination hash or witness, not a human-readable address. Use the version byte and the recipient hash to reconstruct the Bitcoin address.

```typescript theme={"system"}
import bs58check from 'bs58check';
import { bech32, bech32m } from 'bech32';

const hexStringToUint8Array = (hexString: string) =>
  Uint8Array.from(
    hexString.match(/.{1,2}/g)!.map((byte) => parseInt(byte, 16))
  );

function decodeBitcoinAddress(
  version: number,
  recipientHash: string,
  testnet = false,
): string {
  if (version === 1) {
    const prefix = !testnet ? '00' : '6F';
    const bytes = hexStringToUint8Array(
      prefix + recipientHash.replace('0x', '').slice(0, 40)
    );
    return bs58check.encode(bytes);
  }
  if (version === 2) {
    const prefix = !testnet ? '05' : 'C4';
    const bytes = hexStringToUint8Array(
      prefix + recipientHash.replace('0x', '').slice(0, 40)
    );
    return bs58check.encode(bytes);
  }
  const prefix = !testnet ? 'bc' : 'tb';
  if (version === 3) {
    const bytes = hexStringToUint8Array(
      recipientHash.replace('0x', '').slice(0, 40)
    );
    const words = bech32.toWords(bytes);
    words.unshift(0x00);
    return bech32.encode(prefix, words);
  }
  const bytes = hexStringToUint8Array(
    recipientHash.replace('0x', '').slice(0, 64)
  );
  if (version === 4) {
    const words = bech32.toWords(bytes);
    words.unshift(0x00);
    return bech32.encode(prefix, words);
  }
  if (version === 5) {
    const words = bech32m.toWords(bytes);
    words.unshift(0x01);
    return bech32m.encode(prefix, words);
  }

  throw Error(`Unsupported Address Type ${version}`);
}
```

### Constructing the Bitcoin Transaction

Create a transaction with at least one output that exactly matches the order's output description. The transaction can have any number of inputs and additional outputs, allowing for batch filling and consolidation.

If calldata is set on the output, add an `OP_RETURN` output immediately after the filling output:

```typescript theme={"system"}
import * as bitcoin from 'bitcoinjs-lib';

bitcoin.initEccLib(ecc);

const psbt = new bitcoin.Psbt({
  network: mainnet ? bitcoin.networks.bitcoin : bitcoin.networks.testnet,
});

// ... add inputs

psbt.addOutput({ address: to, value: outputValue });

const opReturnData = returnData.replace("0x", "");
if (opReturnData.length > 0) {
  const data_embed = bitcoin.payments.embed({
    data: [hexStringToUint8Array(opReturnData)],
  });
  psbt.addOutput({
    script: data_embed.output!,
    value: 0n,
  });
}

// ... complete transaction
```

***

## Next Steps

<CardGroup cols={2}>
  <Card title="Settling Orders" icon="check" href="/lifi-intents/for-solvers/settlement">
    Finalize orders and claim locked input funds
  </Card>

  <Card title="Collecting Orders" icon="inbox" href="/lifi-intents/for-solvers/orderflow">
    WebSocket and REST order collection
  </Card>

  <Card title="Auctions" icon="gavel" href="/lifi-intents/for-solvers/auctions">
    Limit, Dutch, and exclusive auction types
  </Card>

  <Card title="Reputation" icon="star" href="/lifi-intents/for-solvers/reputation">
    Solver reputation system and registration
  </Card>
</CardGroup>
