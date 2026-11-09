> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Tron vs EVM

> What an integrator or solver must do differently to use LI.FI Intents on Tron compared to an EVM chain: addresses, endpoints, settlement types, and on-chain submission.

Tron is a supported Intents chain (chain ID `728126428`, CAIP-2 `tron:728126428`). Tron runs an EVM-compatible VM, so **inside the protocol the order looks identical to EVM** — the same `StandardOrder` and `MandateOutput` structs, the same 20-byte addresses left-padded to `bytes32`.

The differences are at the edges: how you encode addresses at the HTTP boundary, which contract addresses you talk to, which settlement types exist, and how you submit the origin transaction. This page is a delta reference — for each concern, what EVM does and what Tron does instead.

> [!NOTE]
>
> For general (non-Intents) Tron support across LI.FI products, see the [Tron ecosystem page](https://docs.li.fi/introduction/tron-ecosystem.md).

***

## At a glance

| Concern | EVM | Tron |
| - | - | - |
| Address format — requests | `0x` hex | base58 `T…` (CAIP-2 chain); `0x` 20-byte hex also accepted |
| Address format — responses | `0x` hex | base58 `T…` |
| Settlement types | Escrow **and** Compact | **Escrow only** |
| Order open path | On-chain open **or** gasless `POST /orders/submit` | **On-chain open only** |
| Contract addresses | Deterministic `0x0000…`, identical across EVM chains | **Unique base58**, Tron-specific (see below) |
| Submission library | viem / ethers | **TronWeb** (`TriggerSmartContract` envelope) |
| Origin fee unit | gas (wei / gwei) | `feeLimit` in **SUN** (1 TRX = 1e6 SUN) |
| Cross-chain oracle | Polymer | Polymer (Tron PolymerOracle) |

***

## Quoting

Tron is served by the standard quote endpoint, `POST /api/v1/integrator/quote/request`. It accepts a CAIP-2 chain plus the **native** address, so you pass Tron addresses as base58 (`T…`) and get them back as base58. The full request reference and a Tron example are in [Request a Quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md#tron-request).

The response renders Tron addresses back as base58. The origin transaction for a Tron-origin order is returned under `order.openIntentTx`:

```jsonc theme={"system"}
{
  "quotes": [{
    "order": {
      "openIntentTx": {
        "chain": "tron:728126428",
        "to": "TRV3PsTLRiWpY6sWi5UAvB7Tacb2FLtCNq", // InputSettler (base58)
        "data": "0x7515fd56…",                       // ABI calldata (selector + args)
        "feeLimit": "150000000"                       // SUN cap (≈150 TRX)
      }
    },
    "preview": {
      "inputs":  [{ "chain": "tron:728126428", "user": "TYYUrA6nV1EYxQqo7RiBJh6roFq4mtiBLT", "asset": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t", "amount": "10000000" }],
      "outputs": [{ "chain": "eip155:8453", "receiver": "0x841F63697cFa0e3B54c4D42b3d679F07F7F2485f", "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "amount": "9986765" }]
    }
  }]
}
```

The reverse direction (EVM → Tron) works the same way — put the Tron leg in `outputs` with `chain: "tron:728126428"` and base58 addresses.

If you instead use the legacy V0 `POST /quote/request` endpoint (EIP-7930), the interoperable address must encode the `tron` CAIP-2 namespace and embed the **20-byte** form of the Tron address (see [Address format](#address-format-at-the-boundary)) — and `supportedTypes` there takes the settlement-type vocabulary (`["oif-escrow-v0"]` only on Tron), not `oif-user-open-v0`. Prefer the standard endpoint, or build EIP-7930 with the [`interoperable-addresses`](https://www.npmjs.com/package/interoperable-addresses) library rather than hand-encoding it.

***

## Address format at the boundary

A Tron base58 address (`T…`) is a `0x41` version byte plus the 20-byte account, wrapped in base58check. **The protocol only ever sees the 20-byte account** — the `0x41` version byte and the base58 checksum exist only in the human-facing envelope.

* **Quote endpoint (V1):** pass base58 (`T…`). `0x`-prefixed 20-byte hex is also accepted. Responses always render base58.
* **Legacy V0 endpoint (EIP-7930):** the embedded address is the 20-byte form (base58 decoded, `0x41` + checksum stripped).
* **On-chain / in the order struct:** 20-byte hex, left-padded to `bytes32` — exactly like EVM.

Convert between the two forms with `bs58check`:

```ts theme={"system"}
import bs58check from 'bs58check';

const TRON_VERSION_BYTE = 0x41;

// Tron base58 (T…) -> 20-byte hex used in the protocol / order struct
function tronBase58ToHex20(address: string): `0x${string}` {
  const decoded = bs58check.decode(address); // 21 bytes: 0x41 + 20-byte account
  const body = decoded.slice(1);             // drop the 0x41 version byte
  return ('0x' +
    Array.from(body, (b) => b.toString(16).padStart(2, '0')).join('')
  ) as `0x${string}`;
}

// 20-byte hex -> Tron base58 (T…)
function hex20ToTronBase58(hex: string): string {
  const body = Uint8Array.from(
    hex.replace(/^0x/, '').match(/.{1,2}/g)!.map((b) => parseInt(b, 16)),
  );
  const payload = new Uint8Array(21);
  payload[0] = TRON_VERSION_BYTE;
  payload.set(body, 1);
  return bs58check.encode(payload);
}

// TRV3PsTLRiWpY6sWi5UAvB7Tacb2FLtCNq <-> 0xaa2e58aa1a4107dc8cc7ef41b97be90b25b5b842
```

***

## Wire format is identical to EVM

Once an address is in 20-byte hex form, **nothing downstream changes**. The `StandardOrder` and `MandateOutput` structs, the `bytes32` left-padding (`tokenId`/`oracle`/`settler`/`token`/`recipient`), and the calldata encoding are the same on Tron as on EVM. The `0x41` version byte never appears on-chain. If your EVM order-construction code already produces the correct struct, the only change for Tron is converting addresses at the boundary and submitting via TronWeb.

***

## Settlement: Escrow only

Tron supports only the **Escrow** input settler. There is **no Compact / The Compact / resource-lock** flow on Tron. Consequences:

* On the legacy V0 `POST /quote/request` endpoint, request `["oif-escrow-v0"]` only — never `oif-resource-lock-v0`.
* The [Compact Orders](https://docs.li.fi/lifi-intents/intents-api/compact-orders.md) guide does not apply to Tron.

## Opening orders: on-chain only

Tron orders are opened **on-chain** by calling the Escrow input settler directly with the `openIntentTx` from the quote. The order server and solvers detect the resulting on-chain event automatically.

`POST /orders/submit` (off-chain / gasless submission) is **EVM-only** — there is no gasless Tron path.

***

## Contract addresses

These are **not** the deterministic `0x0000…` addresses used on EVM chains. Tron has no keyless CREATE2 factory, so the deployments live at unique, Tron-specific addresses.

> [!WARNING]
>
> The "same address across all supported chains" statement elsewhere in these docs applies to **EVM chains only**. Do not send funds to the EVM `0x0000…` addresses on Tron — they do not exist there.

| Contract | Base58 (Tron) | Hex (20-byte) |
| - | - | - |
| InputSettler (Escrow) | `TRV3PsTLRiWpY6sWi5UAvB7Tacb2FLtCNq` | `0xaa2e58aa1a4107dc8cc7ef41b97be90b25b5b842` |
| OutputSettler | `TLwJhhq7fExHWdJQfMnsSY7VcreipmhLRm` | `0x784d4f7b6e99b22d923ec99edbe2e11b38ceac93` |
| PolymerOracle | `TPXQkHcGwEdH4Ss8kT4cDxgXt3L4n4zSHJ` | `0x94b0c01e26aff5a6a0fd767afe0e3ca0f8b34e3d` |

***

## On-chain submission (TVM)

The quote's `openIntentTx.data` is full ABI calldata (function selector + arguments) as a `0x`-hex string — it is **not** a ready-to-broadcast EVM transaction. A Tron client must wrap it in a `TriggerSmartContract` envelope, which is TAPOS-bound (it references a recent block via `ref_block_*` / `expiration`), then sign and broadcast it.

* `openIntentTx.to` is the settler **base58** address.
* `openIntentTx.feeLimit` is a **cap on energy spend in SUN**, not an estimate.
* Token approvals use **TRC-20** (not ERC-20).

```ts theme={"system"}
import { TronWeb } from 'tronweb';

const tronWeb = new TronWeb({ fullHost: 'https://api.trongrid.io' /*, privateKey */ });
const { to, data, feeLimit } = quote.order.openIntentTx;

// 1. Approve the InputSettler to pull the TRC-20 input (e.g. USDT)
//    via the token contract's `approve(spender, amount)`.

// 2. Submit the open() calldata wrapped in a TriggerSmartContract envelope.
const tx = await tronWeb.transactionBuilder.triggerSmartContract(
  to,                       // settler (base58)
  '',                       // selector empty: full calldata is supplied via `input`
  { feeLimit, input: data.replace(/^0x/, '') },
  [],
  tronWeb.defaultAddress.base58,
);
const signed = await tronWeb.trx.sign(tx.transaction);
await tronWeb.trx.sendRawTransaction(signed);
```

After the order opens, track it the same way as an EVM order via the [order status API](https://docs.li.fi/lifi-intents/intents-api/track-status.md).

***

## Oracle

For **cross-chain** Tron outputs, the order uses **Polymer**; `MandateOutput.oracle` is the `bytes32` of the Tron PolymerOracle (`TPXQkHcGwEdH4Ss8kT4cDxgXt3L4n4zSHJ`). For **same-chain** (Tron → Tron) intents, the OutputSettler acts as the oracle, as described in [Same-chain intents](https://docs.li.fi/lifi-intents/architecture/overview.md#same-chain-intents). See [Oracle Systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md#polymer) for how Polymer proofs work.

***

## For solvers

Filling a Tron output works like an EVM fill: call `fillOrderOutputs` on the Tron `OutputSettler`. The one Tron-specific step is to submit through **TronWeb**, decoding addresses with the 20-byte rule above.

See [Solving LI.FI Intents](https://docs.li.fi/lifi-intents/for-solvers/intro.md#filling-intents) for the general fill flow.

***

## Next Steps

> [card] [Request a Quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md)
>
> Exact-input/output quoting and the Tron request example

> [card] [Create and Submit Orders](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md)
>
> Construct a StandardOrder and open it on-chain

> [card] [Oracle Systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md)
>
> Polymer and the other verification systems

> [card] [Solving LI.FI Intents](https://docs.li.fi/lifi-intents/for-solvers/intro.md#filling-intents)
>
> How solvers deliver outputs


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.