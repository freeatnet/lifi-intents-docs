> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Filling Orders

> Deliver output assets on EVM chains to fulfill LI.FI intents.

To finalize an intent and claim the locked input funds, the solver must deliver all outputs described in the order. This page covers EVM filling.

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
