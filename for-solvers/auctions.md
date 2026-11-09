> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Auctions

> Decode limit, Dutch, and exclusive auction contexts before filling an order.

LI.FI Intents support four order types: Simple limit orders, Dutch auctions, Exclusive limit orders, and Exclusive dutch auctions. Each intent output encodes a specific order type. Generally, all outputs should be of the same order type.

For an order with multiple outputs, the solver of the first output will **always** be the recipient of the inputs. The inputs cannot be claimed until all outputs are filled. The *auction* happens on-chain, with the winner determined by speed.

> [!WARNING]
>
> Because the first-output solver owns the order, they control *when* the remaining outputs get filled — and for time-priced outputs (Dutch auctions) they can wait until the price bottoms out. Intent issuers should avoid Dutch order types on non-first outputs, or add exclusivity for a trusted solver.

Most LI.FI intents are [exclusive limit orders](#exclusive-limit-order) — that is what the LI.FI widget and most other integrators issue.

### Limit Order (First come, First serve)

LI.FI intent limit orders are simple: single price, first execution. That means the amount set in `MandateOutput` is final and will be what you are paying. The output can be filled once per `orderId` (the output is hashed and stored in a map with `orderId`).

The winner of an order is the first solver to call `fill(...)` and set their identifier for the output.

You can identify Limit Orders by `output.context` of either `0x` or the order type `0x00`.

### Exclusive Limit Order

Exclusive limit orders work like limit orders — single price, first execution — with one difference: the context embeds a cutoff timestamp, and until that time only one named solver may fill the order. This type of intent is often used in conjunction with the LI.FI order Server [Reputation](https://docs.li.fi/lifi-intents/for-solvers/reputation.md) to eliminate wasted gas on failed transactions.

You can identify Exclusive Limit Orders by their order type `0xe0`. The context is encoded as: `bytes1(0xe0) | bytes32(exclusiveFor) | bytes4(startTime)`, with `startTime` being the time when anyone can fill the order.

Intent issuers should usually set the exclusivity period to 30–60 seconds, though it can be shorter or longer.

### Dutch Auction

LI.FI intent Dutch auctions are capped dutch auctions. They are fixed in price before and after certain timestamps. The slope, startTime, and stopTime are encoded in `output.context`.

The winner of an order is the first solver to call `fill(...)` and set their identifier for the output. Winning works the same as a limit order, except that the earlier you fill, the more you pay.

You can identify Dutch Auctions by the order type: `0x01`. The context is encoded as: `bytes1(0x01) | uint32(startTime) | uint32(stopTime) | uint256(slope)`, with the final amount being computed as:

```solidity theme={"system"}
uint32 currentTime = max(block.timestamp, startTime);
if (stopTime < currentTime) return amount;
uint256 timeDiff = stopTime - currentTime;
return amount + slope * timeDiff;
```

The maximum amount of the auction is `amount + (stopTime - startTime) * slope`.

You can find the on-chain [implementation here](https://github.com/openintentsframework/oif-contracts/blob/222989b15241ec680d2a0503c7396e1b6c649a50/src/output/simple/OutputSettlerSimple.sol#L94-L118).

### Exclusive Dutch Auction

LI.FI intent exclusive dutch auctions are very similar to LI.FI intent dutch auctions. However, before the auction slope begins the order is exclusive to a specific solver like [Exclusive Limit Order](#exclusive-limit-order).

You can identify Exclusive Dutch Auctions by their order type `0xe1`. The context is encoded as `bytes1(0xe1) | bytes32(exclusiveFor) | uint32(startTime) | uint32(stopTime) | uint256(slope)`.

The exclusivity window sits before the Dutch slope begins, so an exclusive solver who fills during it pays the auction's maximum price.

### Solana Encoding Notes

Solana's `output_settler_simple` supports the same four order types with the same type bytes (`0x00`, `0x01`, `0xe0`, `0xe1`) and the same payload layouts, with all integers big-endian. The exact context sizes are tabulated in [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#order-types).

Three Solana error names to know: an unknown type byte fails with `NotImplemented`, a wrong payload length with `InvalidContextDataLength`, and a fill during someone else's exclusivity window with `ExclusiveTo`.

> [!NOTE]
>
> **Solana `u64` limit on `slope` and the resolved amount.** Although `slope` is encoded as 32 bytes (matching the EVM `uint256` field), the Solana settler narrows it to a `u64`: the upper 24 bytes must be zero. The computed output amount must also fit in a `u64` — SPL token amounts are `u64`.
>
> This differs from the EVM settler, which treats `slope` as a full `uint256` and returns the minimum amount once `stopTime` has passed regardless of slope size. A Dutch (or Exclusive Dutch) order signed with `slope > u64::MAX` — or whose resolved amount exceeds `u64::MAX` — is rejected when parsed on Solana and can never be filled, even after the auction bottoms out. When constructing cross-chain Dutch orders destined for Solana, keep both `slope` and the maximum resolved amount within the `u64` range.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.