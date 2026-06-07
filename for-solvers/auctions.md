> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Auctions

> LI.FI Intents powers on-chain auctions and flexible order types, letting you compete for fills and optimize execution across cross-chain settlement.

LI.FI Intents support four order types: Simple limit orders, Dutch auctions, Exclusive limit orders, and Exclusive dutch auctions. Each intent output encodes a specific order type. Generally, all outputs should be of the same order type.

For an order with multiple outputs, the solver of the first output will **always** be the recipient of the inputs. The inputs cannot be claimed until all outputs are filled. The *auction* happens on-chain, with the winner determined by speed.

The primary order type of LI.FI intents is [Exclusive Limit Order](#exclusive-limit-order) which is the one served from the LI.FI widget and most other integrators.

### Limit Order (First come, First serve)

LI.FI intent limit orders are simple: single price, first execution. That means the amount set in `OutputDescription` is final and will be what you are paying. The output can be filled once per `orderId` (the output is hashed and stored in a map with `orderId`).

The winner of an order is the first solver to call `fill(...)` and set their identifier for the output.

You can identify Limit Orders by `output.context` of either `0x` or the order type `0x00`.

### Exclusive Limit Order

A LI.FI intent exclusive limit orders are very similar to a LI.FI intent limit orders. Single price, first execution. **Except** it has a cutoff timestamp embedded that only allows a specific solver to fill the intent before. This type of intent is often used in conjunction with the LI.FI order Server [Reputation](/lifi-intents/for-solvers/reputation) to eliminate wasted gas on failed transactions.

You can identify Exclusive Limit Orders by their order type `0xe0`. The context is encoded as: `bytes1(0xe0) | bytes32(exclusiveFor) | bytes4(startTime)`, with `startTime` being the time when anyone can fill the order.

For intent issuers, it is recommended to set the exclusivity period to between 30 seconds to 1 minute but it may be shorter or longer.

### Dutch Auction

LI.FI intent Dutch auctions are capped dutch auctions. They are fixed in price before and after certain timestamps. The slope, startTime, and stopTime are encoded in `output.context`.

The winner of an order is the first solver to call `fill(...)` and set their identifier as the output. Notice that the win selection is equivalent to the limit order, except the earlier the fill is called, the more the solver pays.

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

You can identify Exclusive Dutch Auctions by their order type `0xe1`. The context is encoded as `bytes1(0x01) | bytes32(exclusiveFor) | uint32(startTime) | uint32(stopTime) | uint256(slope)`.

It is important to clarify that the exclusivity window is before the dutch auction slope begins, thus if the intent is filled by the exclusive solver they will be paying the maximum price for the auction.
