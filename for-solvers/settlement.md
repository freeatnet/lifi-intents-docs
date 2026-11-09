> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Settling Orders

> Relay fill proofs and finalize orders to claim the locked input funds.

Settling a LI.FI Intents order takes up to two transactions:

1. Submit the message validation proof.
2. Finalize the order on the resource lock.

## Oracle Validation

How you relay a proof depends on the order's oracle system. Follow the instructions for that system in [Oracle Systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md).

## Finalizing Orders

Once you see the `OutputProven` event, the output has been validated and you can call `finalise`.

```
event OutputProven(uint256 chainid, bytes32 remoteIdentifier, bytes32 application, bytes32 payloadHash);
```

The `finalise` ABI depends on the settler: `InputSettlerCompact` requires signatures, `InputSettlerEscrow` does not.

For `InputSettlerCompact`, submit the `StandardOrder`, the attached signatures as `bytes.concat(sponsor, allocator)`, the timestamps of the fills (which you can read from the `OutputFilled` event), and the solver's identifier. If there are multiple outputs, always index lists by their position in the order. Then call `finalise`. The flow is the same for `InputSettlerEscrow`, except that you provide no signatures.

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

> [!NOTE]
>
> If an order contains multiple outputs and two solvers filled different outputs, then the solver of the first output is the canonical solver.

### Signature Finalise

The solver must be the caller of `finalise`, as they can redirect the funds and modify the optional callback. For some solvers this is not possible — for example, when the solver identifier is set to a pool. In that case, use `finaliseWithSignature` with an EIP-712 `AllowOpen` signature:

```solidity theme={"system"}
struct AllowOpen {
    bytes32 orderId;
    bytes32 destination;
    bytes call;
}
```

Anyone holding the signature can then call the input settler's `finalise` with the signed destination and, if one is present, the signed call.

### Callback

Some solvers may use callbacks for configuring received funds. The `call` parameter of `finalise[WithSignature]` forwards the provided payload to `destination` via `orderFinalised`:

```solidity theme={"system"}
function orderFinalised(
    uint256[2][] calldata inputs,
    bytes calldata call
) external
```

The `orderFinalised` call must not fail.

For a same-chain intent, `orderFinalised` runs after the inputs are paid to `destination` but before the output proofs are validated. This allows solvers to collect intent inputs before filling the intents in an atomic transaction.

***

## Settling on Solana

The standard Solana path validates any cross-chain proofs, then calls `finalise` on [`input_settler_escrow`](https://docs.li.fi/lifi-intents/architecture/solana-programs#settlement). Proof loading and validation may require multiple transactions.

An order opened on Solana always has exactly one output — `open` rejects larger orders on transaction size. A same-chain order that also meets the input-first requirements can instead use [input-first settlement](#input-first-solana-settlement).

> [!WARNING]
>
> The solver id recorded by a fill is the identity that claims the inputs on the origin chain — not necessarily the destination-chain transaction signer. For a Solana-origin order, output 0 must record the raw bytes of the Solana account that will sign `finalise` — a key you control, or a program PDA that signs through CPI. The compact same-chain fills are the exception: they record their own signer, and must be called top level.

Build `solve_params` from the output's `OutputFilledEvent`, using the event's exact `solver` and `timestamp`. Pass the corresponding `LocalAttestation` (same-chain) or Polymer-created `Attestation` (remote) in `remaining_accounts`, followed by its rent-refund slot.

### Finalization accounts and checks

```rust theme={"system"}
pub fn finalise(
    ctx: Context<Finalise>,
    order: StandardOrder,
    solve_params: Vec<SolveParams>,
) -> Result<()>
```

Only the canonical solver — the solver of output 0 — can call it. The program enforces both the length and the signer:

```rust theme={"system"}
require!(
    solve_params.len() == order.outputs.len(),
    InputSettlerEscrowErrors::SolversLengthMismatch
);
require!(
    solve_params[0].solver == ctx.accounts.solver.key().to_bytes(),
    InputSettlerEscrowErrors::FirstSolverNotInSolvers
);
```

The accounts you resolve as the caller:

* `solver` — you, the transaction signer (the output-0 solver)
* `destination` and its token account — where the escrowed inputs go; because you choose it at finalization time, the canonical solver must be the caller
* `sponsor` — the account that paid for `open` (read it from `order_context.sponsor`); receives the rent refund, or you do if the sponsor is an executable account
* `order_context` and its escrow ATA — derived from the `order_id`
* `local_consumer` — the escrow's `["local_consumer"]` PDA (`BZfvceVEFY1SYswSRepgfDCbMXTtWfTwwsRXKcTCBVx8`), followed by the `intents_protocol` program
* `remaining_accounts` — the attestation, then its rent-refund slot: `[attestation, rent_refund]`. Passing only the attestation fails with `WrongAttestationCount`.

Which attestation account depends on each output's chain:

* **Same-chain output** (`output.chain_id` equals the Solana chain id): pass the fill's `LocalAttestation`. No oracle is involved. Under LI.FI's canonical encoding the escrow does not consume this attestation, so the rent-refund slot is ignored — pass any existing read-only account, such as the system program.
* **Cross-chain output**: run the oracle flow so the input-side oracle creates an `Attestation` PDA, then pass that. The rent-refund slot is ignored here too. See [Oracle Validation](#oracle-validation) above and [Attestations](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#attestations) for the account model.

The instruction takes N attestations followed by N rent-refund slots for N outputs, but Solana-origin orders have one output, so in practice it is always `[attestation, rent_refund]`. The full account list is in the [input\_settler\_escrow IDL](https://docs.li.fi/idl/input_settler_escrow.json), and every `remaining_accounts` layout is in the [Solana IDL bundle](https://docs.li.fi/lifi-intents/architecture/solana-idl.md#remaining-accounts).

### Input-first Solana settlement

Use this route for a single-output Solana → Solana escrow order with empty `callback_data`, a limit or exclusive-limit context, and the output-settler state PDA in all three identity fields.

The transaction releases the input before delivering the output. Your intermediate instructions can convert that input into the required output.

The transaction carries these top-level instructions, in this order:

```text theme={"system"}
input_settler_escrow::finalise_with_prefill(order_id)
  → your instructions: swap, mint, burn, or another strategy
  → output_settler_simple::fill_samechain({ order_id, order })         // SPL or Token-2022 output
    or output_settler_simple::native_fill_samechain({ order_id, order }) // native SOL output
```

`finalise_with_prefill` scans the [Instructions sysvar](https://docs.rs/solana-program/latest/solana_program/sysvar/instructions/index.html) for exactly one matching compact fill later in the transaction. It verifies the reconstructed order and fill accounts before releasing the escrow.

If the fill or an intermediate instruction fails, the entire transaction rolls back.

> [!NOTE]
>
> "Compact" here describes the smaller instruction payload, not The Compact. Solana still has one input settler — the escrow — and no resource locks.

#### Build the compact fill

Pass the `order_id` and six order fields: `nonce`, `expires`, `fill_deadline`, input amount, output amount, and pricing context. The programs reconstruct the remaining fields:

| Rebuilt field | Read from |
| - | - |
| `user`, input mint | The order's escrow context on the input side |
| `origin_chain_id`, `output.chain_id` | The settler state PDAs |
| `output.recipient`, `output.token` | The fill's recipient and mint accounts (32 zero bytes for a native SOL fill) |
| `exclusive_for` in `output.context` | The compact payload, or the fill's signer when you use the shorthand |
| `input_oracle`, `output.oracle`, `output.settler` | Fixed to the output-settler state PDA |
| `output.callback_data` | Always empty |

Use an empty context for a limit order. For an exclusive limit, use `0xe0 || exclusive_for || start_time`. Use the compact shorthand only when your signing key is the order's `exclusive_for` value.

> [!WARNING]
>
> * **Keep both settlement instructions top level.** Calling either through CPI fails. Include exactly one matching compact fill, after `finalise_with_prefill`.
> * **Use canonical associated token accounts.** Your output account must already exist and hold the output amount before the fill runs. Native SOL comes from your signing account.
> * **Fund account creation.** You may pay rent for the input destination, the recipient token account, and the `fill_record`. The compact fill creates no attestation. You can reclaim the `fill_record` rent with `close_fill_record` once `fill_deadline` + 48 hours has passed.
> * **Never send the compact fill alone.** Without the matching `finalise_with_prefill` before it in the same transaction, the output is still paid but nothing proves the fill, so you cannot claim the input afterwards.
> * **Fit everything into one transaction.** Solana transactions are limited to 1,232 bytes. Use a versioned transaction and a previously created [address lookup table](https://solana.com/docs/advanced/lookup-tables) when needed.
> * **Submit by `fill_deadline`.** The deadline is inclusive; a later fill rolls back the transaction.
> * **Watch `FinalisedEvent`.** This route emits the same event as `finalise`; its `settler` field is the order's `order_context` PDA. It carries no amounts — read them from the escrow's `transfer_checked` inner instruction. Indexers must ignore events from failed transactions.

See [Input-first settlement errors](https://docs.li.fi/lifi-intents/knowledge-database/solana-debugging.md#input-first-settlement-errors) for failure codes and common fixes.

The full verification model is in [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#input-first-settlement-finalise_with_prefill).

### Solana Refunds

Two permissionless refund paths return the escrow to the user, and both permanently close the order:

* From `expires` onward, anyone can call `refund(order)`.
* Earlier, once `fill_deadline` passes and one output is proven not filled, anyone can call `refund_on_non_fill(order, output_index)` — no need to wait for expiry. It takes the escrow's `local_consumer` PDA and `remaining_accounts = [attestation, rent_refund]`, like `finalise`. For a Solana output, the not-filled proof (`attest_not_filled`) can only be created within 48 hours after `fill_deadline`. If the window is missed, the order refunds at `expires`.

> [!WARNING]
>
> Finalize promptly. Once a refund lands, your repayment is gone, even if you filled some outputs — finalization and the two refunds exclude each other. Fill every output before `fill_deadline` and finalize well before `expires`.

Both paths are documented in [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#refunds).


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.