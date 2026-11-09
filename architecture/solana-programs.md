> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Solana programs

> Reference for the Anchor programs behind LI.FI Intents on Solana: order structs, the escrow input settler, the output settler, and the attestation model that proves fills.

On Solana, LI.FI Intents runs as a set of [Anchor](https://www.anchor-lang.com/docs) programs. The design mirrors the EVM architecture — input settlement, output settlement, oracle validation — but the moving parts are Solana-native: funds sit in token accounts owned by PDAs, and proofs are PDA accounts anyone can recompute from public inputs.

This page is the program reference: the order structs, the settlement lifecycle of each program, and the proof and attestation model. If terms like PDA or CPI are new to you, start with the [Solana vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md) primer.

## The four deployed programs

The current deployment splits the system into four programs, each with one job. The same program IDs are used on devnet and mainnet-beta.

| Program | Program ID | Role |
| - | - | - |
| `input_settler_escrow` | `LiFiRp8RM7nJUZyUYC9FPPpDr7sAy5XPfBN6ABzBgT7` | Holds the user's input funds in escrow on the origin chain until the order is finalized or refunded |
| `output_settler_simple` | `LiFiEDFjz5x1jJe9gSXNDHQW4dWt4yLXdp2VN4EiQUt` | Delivers outputs to recipients on the destination chain and records each fill |
| `polymer` | `LiFiBtfyPT1DnTHTAeZ2rwr5RgMrThwA5kt7KGT5nBV` | Carries fill proofs from the output chain back to the input chain |
| `intents_protocol` | `LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK` | Stores attestation accounts and chain configuration that the other programs rely on |

![Overview of the LI.FI intent programs on Solana](https://mintcdn.com/lifi/OS19xH0Jt9xNmdgc/images/lifi-intent-svm-overall.png?fit=max\&auto=format\&n=OS19xH0Jt9xNmdgc\&q=85\&s=3428f2da570512a578dcb974255b6732)

The interface reference for each program is its Anchor IDL — see [Program IDLs](#program-idls) below.

Whether a field takes a **state PDA** or a **program ID** differs per field and per route — see [Which identity goes in which field](#which-identity-goes-in-which-field) below.

### Program IDs and derived accounts

A program ID identifies executable code. Most instructions also require singleton program-derived accounts (PDAs) that hold the deployed program's state. These addresses are deterministic and are the same on devnet and mainnet-beta because both clusters use the same program IDs and seeds; their stored configuration is cluster-specific.

| Account | PDA seed | Address | Where it is used |
| - | - | - | - |
| `ChainId` | `"chain_id"` under `intents_protocol` | `BkEw3WHFvJR9a5deUcwPLJ79yK3r9YGdQ61TehFXzAQ` | Canonical chain configuration used by all four programs |
| `InputSettlerEscrowAccount` | `"input_settler_escrow"` under `input_settler_escrow` | `Cj3mSoPJtgi5bubC9rLz8oM1DuXoJj97RMw1uC6ev9zm` | The `input_settler_escrow` account passed to open, finalization, and refund instructions |
| `OutputSettlerSimpleAccount` | `"output_settler_simple"` under `output_settler_simple` | `DHShHmVkTwCzUzAQbCu4GDqJmursuDscNR6o4hTBgeRy` | The settler instruction account and the value encoded as `MandateOutput.settler` for a Solana output |
| `OraclePolymer` | `"polymer"` under `polymer` | `49zLKETMq34CUC2E2wL1xvv6uN2AUgyhjVX221mjE3Rw` | The oracle instruction account and the `StandardOrder.input_oracle` for Polymer-backed Solana inputs |
| Escrow `local_consumer` | `"local_consumer"` under `input_settler_escrow` | `BZfvceVEFY1SYswSRepgfDCbMXTtWfTwwsRXKcTCBVx8` | Data-less signing authority passed to `finalise` and `refund_on_non_fill`; consumes local attestations committed to the escrow program |
| Polymer `local_consumer` | `"local_consumer"` under `polymer` | `2m1udXnYmaxjSXBBE88yZZcwrtT127DqLcwK9B3WCAus` | Data-less signing authority passed to `submit_consume`; consumes local attestations committed to the Polymer program |

Anchor clients can resolve these PDAs from the seed metadata in the published IDLs. Other clients can derive them locally with `PublicKey.findProgramAddressSync([Buffer.from(seed)], programId)`. A plain RPC lookup cannot infer an unknown program's PDA seeds.

The Polymer PDA stores its configured prover program ID, currently `CdvSq48QUukYuMczgZAVNZrwcHNshBdtqrjW26sQiGPs` on both clusters. Read that field from the `OraclePolymer` account instead of duplicating it in order data. Chain mappings, order contexts, fill records, token accounts, and attestations depend on a chain or order and are derived as needed; they are not deployment addresses. Order contexts and fill records must be derived client-side — see [Program IDLs](#program-idls).

> [!NOTE]
>
> Wormhole also has a mainnet deployment; see the [deployment overview](https://docs.li.fi/lifi-intents/architecture/overview.md#solana). The bundled Wormhole IDL is a source-interface reference with a different embedded address; do not use that address as the mainnet deployment address.

### Which identity goes in which field

Every 32-byte identity slot in an order names a specific account, and program IDs and state PDAs are not interchangeable. Each rule applies to a specific leg: the `MandateOutput` rows below hold when that **output is on Solana** (an EVM output carries EVM identities instead), and the `input_oracle` row holds when the **order is opened on Solana**.

| Order field | Applies when | Correct value | When it is checked | If it is wrong |
| - | - | - | - | - |
| `MandateOutput.settler` | The output is on Solana | Output-settler **state PDA** (`DHShHmVkTwCzUzAQbCu4GDqJmursuDscNR6o4hTBgeRy`) | At fill, against the settler account passed to the instruction | The fill is rejected (`InvalidOutputSettler`); the order can only expire and refund |
| `MandateOutput.oracle` | The output is on Solana and the order is cross-chain | Polymer **program ID** (`LiFiBtfyPT1DnTHTAeZ2rwr5RgMrThwA5kt7KGT5nBV`) | Not checked at fill; checked later by the oracle's `submit` or `submit_consume` as the attestation consumer | **The fill succeeds**, but proof submission fails and the solver cannot claim repayment |
| `StandardOrder.input_oracle` | The order is opened on Solana and is cross-chain | Polymer **state PDA** (`49zLKETMq34CUC2E2wL1xvv6uN2AUgyhjVX221mjE3Rw`) | At finalization, as the attestator of the expected `Attestation` account | Finalization never finds the attestation; the escrow is stuck until refund |

For a same-chain Solana → Solana order, no oracle program participates, and one rule applies: **all three slots — `input_oracle`, `output.oracle`, and `output.settler` — hold the output-settler state PDA** (`DHShHmVkTwCzUzAQbCu4GDqJmursuDscNR6o4hTBgeRy`). LI.FI's builder and tooling encode same-chain orders this way. It is the identity [input-first settlement](#input-first-settlement-finalise_with_prefill) requires, because that route rebuilds all three slots from the settler itself. The same order also settles through the standard `fill` → `finalise` route.

> [!NOTE]
>
> On the standard route, the fill's `LocalAttestation` names `output.oracle` as its consumer. With the canonical encoding that consumer is the output settler rather than the escrow, so `finalise` validates the attestation but does not consume it, and its rent (about 0.0012 SOL, paid by the filler) is not reclaimed. Setting `output.oracle` to the `input_settler_escrow` program ID would let `finalise` consume the attestation and return that rent, but the order would then no longer be eligible for input-first settlement. LI.FI uses the canonical encoding.

> [!WARNING]
>
> `open` does not validate these identity fields, so an order with a wrong value is accepted and escrows funds anyway. Solvers must verify all three identities against this page and **reject** any mismatch before filling. A wrong `output.settler` at least makes the fill fail. A wrong `output.oracle` on a cross-chain order does not: the fill instruction never compares that field against an account, so the fill succeeds as signed. Proof submission then fails, and the solver cannot claim the escrowed inputs.

Re-derive these values rather than trusting any 32-byte constant on shape alone: resolve the program IDs from the table above and derive each PDA from its documented seeds, or pin the published addresses as golden values in your integration tests.

## Order structure

Every Solana intent is a `StandardOrder`. The user commits one input on Solana and describes the output that solvers must deliver — on Solana or on an EVM chain.

> [!NOTE]
>
> The struct carries a `Vec` of outputs, but an order opened on Solana has **exactly one output** in practice: `open` rejects any order whose projected `finalise` transaction would not fit, and two outputs already exceed the limit (see [Opening](#opening)). Orders that originate on another chain can still send several outputs to Solana.

```rust theme={"system"}
pub struct StandardOrder {
    pub user: Pubkey,
    pub nonce: u128,
    pub origin_chain_id: u128,
    pub expires: u32,
    pub fill_deadline: u32,
    pub input_oracle: Pubkey,
    pub input: MandateInput,
    pub outputs: Vec<MandateOutput>,
}
```

The input is a single token and amount on Solana:

```rust theme={"system"}
pub struct MandateInput {
    pub token: Pubkey,  // SPL mint
    pub amount: u64,
}
```

Each output uses 32-byte fields so it can describe any destination chain. Solana addresses are the raw 32 bytes of the pubkey; EVM addresses are left-padded with zeros:

```rust theme={"system"}
pub struct MandateOutput {
    pub oracle: [u8; 32],      // output-side oracle identifier
    pub settler: [u8; 32],     // output settler identifier
    pub chain_id: [u8; 32],    // u128, big-endian, padded to 32 bytes
    pub token: [u8; 32],
    pub amount: [u8; 32],
    pub recipient: [u8; 32],
    pub callback_data: Vec<u8>,
    pub context: Vec<u8>,      // order type and auction parameters
}
```

The order's identity comes from its content: `order_id = keccak256(borsh_serialize(StandardOrder))`. Any party can recompute it, which is what makes the PDA-based bookkeeping below work. The full type definitions, including Borsh layouts, are in the [program IDLs](#program-idls).

> [!NOTE]
>
> Although `MandateOutput.amount` and the Dutch-auction `slope` travel as 32-byte values, Solana narrows them to `u64` at fill time. A value above `u64::MAX` makes the output unfillable (`AmountTooLarge` / `ArithmeticOverflow`).

## Input settlement: `input_settler_escrow`

The input settler is an escrow. Opening an order moves the user's tokens into an escrow account; they leave it exactly once — to the solver at `finalise` or `finalise_with_prefill`, or back to the user at `refund` or `refund_on_non_fill`. All four paths close the order's context PDA, so once one succeeds the others can never execute.

### Program state

Three account types track an order's life:

```rust theme={"system"}
/// PDA("input_settler_escrow") — singleton, created by initialize().
pub struct InputSettlerEscrowAccount {
    pub chain_id: u128,
    pub bump: u8,
}

/// PDA("order_context", order_id) — owns the escrow token account.
/// Created at open, closed at finalise / finalise_with_prefill / refund /
/// refund_on_non_fill. Rent goes to the sponsor (see "Rent returns" below).
pub struct OrderContext {
    pub input_token: Pubkey,
    pub user: Pubkey,
    pub sponsor: Pubkey,
    pub bump: u8,
}

/// PDA("consumed_order", order_id) — empty replay marker, never closed.
pub struct ConsumedOrder {}
```

### Opening

`open(order)` escrows the input: it transfers `order.input.amount` from the user's token account into an associated token account owned by the order's `OrderContext` PDA. The instruction needs two signers — the `user`, who must equal `order.user` and authorizes the token debit, and a `sponsor`, who pays the rent for the order's accounts. The sponsor does not have to be the transaction fee payer, and it can be the same key as the user. The sponsor receives the escrow token account and `order_context` rent back when the order settles; the `consumed_order` rent is never returned — that PDA is a permanent replay guard — so budget it as a per-order cost when sponsoring opens.

Before moving funds, `open` validates the order: `fill_deadline` must not have passed (it may equal the current time), `expires` must be strictly in the future and after `fill_deadline`, the order must have outputs, and the chain id and input mint must match the order. It also estimates the size of the eventual `finalise` transaction and rejects the order (`FinaliseTxTooLarge`) if the estimate exceeds **1142 bytes** — the 1232-byte packet limit minus 90 bytes of headroom for ComputeBudget instructions:

```text theme={"system"}
760 + 302·N_outputs + Σ(|callback_data| + |context|) − (32 if sponsor == user)
```

One output with an empty callback and a limit context projects to 1,062 bytes (1,030 when the sponsor is the user), leaving 80 bytes (112 when the sponsor is the user) for the output's `callback_data` and `context` combined. Two outputs project to at least 1,332 bytes, so a Solana-origin order cannot have more than one output.

A successful `open` emits an `OpenEvent` carrying the `order_id` and the full `StandardOrder` — this event is how the order server and solvers discover on-chain opens.

#### Rent returns

Every settlement path — `finalise`, `finalise_with_prefill`, `refund`, `refund_on_non_fill` — closes the escrow token account and `order_context` and sends both rents to the **sponsor**. If the sponsor account is executable (a program), both rents go to the settlement signer instead: the `solver` for either finalization route, the `signer` for either refund. If that account is also executable, the instruction fails with `ExecutableRentRefundAccount`.

### Settlement

`finalise(order, solve_params)` pays the solver. It verifies that every output was filled, then transfers the escrowed funds to the solver's chosen destination and closes the order's accounts. `solve_params` contains one `{ solver, timestamp }` entry per output, in output order, and the solver of output 0 — the canonical solver for the order — must sign. Besides the escrow accounts, the instruction takes the escrow's [`local_consumer`](#program-ids-and-derived-accounts) PDA and the `intents_protocol` program, which it uses to consume local attestations committed to the escrow.

`remaining_accounts` carries **two entries per output**: first one attestation per output, then one rent-refund account per output, both in output order:

```text theme={"system"}
[attestation_0, …, attestation_(N−1), rent_refund_0, …, rent_refund_(N−1)]
```

* The attestation is a [`LocalAttestation`](https://docs.li.fi/lifi-intents/knowledge-database/glossary#localattestation) for a same-chain output, or an [`Attestation`](https://docs.li.fi/lifi-intents/knowledge-database/glossary#attestation) created by the order's `input_oracle` for a cross-chain one.
* The rent-refund slot matters only for a local attestation whose committed consumer is the escrow program (`output.oracle` = `input_settler_escrow` program ID). Then both accounts must be writable and the refund account must be the attestation's stored `rent_refund`; `finalise` consumes the attestation and returns its rent.
* For every other attestation — remote ones, and same-chain ones using the canonical encoding — the slot is **required but ignored**. Any existing account works, such as the system program, and both entries may be read-only.

A wrong count fails with `WrongAttestationCount`. The transaction shape and solver guidance are in [Settlement](https://docs.li.fi/lifi-intents/for-solvers/settlement.md).

Settlement emits a `FinalisedEvent`. Its `settler` field is the order's `order_context` PDA, not the escrow singleton or the program ID:

```rust theme={"system"}
pub struct FinalisedEvent {
    pub settler: Pubkey,          // the order_context PDA
    pub order_id: [u8; 32],
    pub solver: [u8; 32],
    pub destination: [u8; 32],
}
```

The event carries no amounts. Read the settled amount from the escrow's `transfer_checked` inner instruction. For a Token-2022 transfer-fee mint, the destination receives that amount minus the fee.

### Input-first settlement (`finalise_with_prefill`)

`finalise_with_prefill(order_id)` pays the solver before delivering the output. It supports one-output Solana orders whose compact fields can reconstruct the canonical `StandardOrder`. See [Input-first Solana settlement](https://docs.li.fi/lifi-intents/for-solvers/settlement.md#input-first-solana-settlement) for the solver workflow.

The instruction scans the [Instructions sysvar](https://docs.rs/solana-program/latest/solana_program/sysvar/instructions/index.html) for exactly one later `fill_samechain` or `native_fill_samechain`. The fill must be top level, signed by the same solver, and carry the same `order_id`.

The escrow reconstructs the order from the compact payload and authenticated accounts. It verifies the order hash, the exact account count (10 for `fill_samechain`, 5 for `native_fill_samechain`), the ordered account addresses, the canonical token accounts, the `fill_record` PDA, and the fixed program IDs. Any mismatch fails the transaction and rolls back the escrow release.

No deadline check is needed here. `fill_deadline` is part of `order_id`, and the fill itself refuses a passed deadline, so a fill that executes was on time.

Both settlement instructions must be top level (`NotTopLevel`). The settler does not inspect the instructions between them, so those instructions may swap, mint, or create accounts. The complete transaction must still fit Solana's size, compute, and account-lock limits.

The compact fills do not create a `LocalAttestation`: settlement already happened earlier in the same transaction, so there is nothing left to prove. They create only the `FillRecord`.

> [!WARNING]
>
> A compact fill must follow a matching `finalise_with_prefill` in the **same transaction**. The compact fill handlers do not check for it themselves. A compact fill sent on its own still delivers the output, but it leaves no attestation, so the solver cannot claim the input through `finalise` afterwards.

Settlement emits the same `FinalisedEvent` as `finalise`, with `settler` set to the `order_context` PDA. The escrow sweeps its full live balance, so tokens sent to the escrow after `open` are paid out too; read the amounts from the `transfer_checked` inner instruction.

### Refunds

Escrowed funds always return to `order.user`, through one of two permissionless paths:

* `refund(order)` — anyone can call it once `order.expires` is reached (`now >= expires`).
* `refund_on_non_fill(order, output_index)` — anyone can call it as soon as one output is proven not filled, which can happen right after `fill_deadline` passes. One proven non-fill refunds the whole order, because `finalise` needs every output proven filled. It takes the escrow's `local_consumer` PDA and the `intents_protocol` program, and exactly two `remaining_accounts`: `[attestation, rent_refund]`, with the same consume-or-ignore rules as `finalise`. How the not-filled proof is produced is described under [non-fill attestations](#non-fill-attestations-early-refunds).

A late fill cannot undo a refund: `finalise` rejects fills with `timestamp > fill_deadline` (`FilledTooLate`), and the settlement paths exclude each other through the `order_context` PDA.

## Output settlement: `output_settler_simple`

The output settler delivers outputs on Solana and records each delivery in a `FillRecord`; ordinary fills also create a `LocalAttestation`. It supports classic SPL tokens, [compatible Token-2022 tokens](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md#token-support), and native SOL.

### Order types

Each output's `context` field encodes its order type: the first byte selects the type, the rest is type-specific payload with all integers big-endian.

| Type | First byte | Payload after the type byte | Full context size |
| - | - | - | - |
| Limit | `0x00` (or empty context) | — | 1 byte (or 0) |
| Dutch auction | `0x01` | `u32 start_time \|\| u32 stop_time \|\| bytes32 slope` | 41 bytes |
| Exclusive limit | `0xe0` | `bytes32 exclusive_for \|\| u32 start_time` | 37 bytes |
| Exclusive Dutch auction | `0xe1` | `bytes32 exclusive_for \|\| u32 start_time \|\| u32 stop_time \|\| bytes32 slope` | 73 bytes |

Any other type byte fails with `NotImplemented`; a wrong payload length fails with `InvalidContextDataLength`. The pricing formulas behind these types are shared with EVM — see [Auctions](https://docs.li.fi/lifi-intents/for-solvers/auctions.md).

The compact same-chain fills reach only two of these types: an empty context and the exclusive limit. A Dutch auction, an exclusive Dutch auction, or a limit order written as an explicit `0x00` byte must go through `fill` or `native_fill`.

### Fills

Four instructions deliver outputs. The standard pair takes a full `MandateOutput`; the compact pair takes the smaller payload used by [input-first settlement](#input-first-settlement-finalise_with_prefill).

* `fill(order_id, mandate_output, fill_deadline, filler_data)` — classic SPL and compatible Token-2022 outputs. It transfers the tokens to the recipient, creates the `FillRecord` and `LocalAttestation` accounts, and emits an `OutputFilledEvent`. The `order_id` is the input-chain identifier; it is not validated at fill time.
* `native_fill(...)` — native-SOL outputs. The output token must be the zero address `[0; 32]`; the transfer goes through the system program.
* `fill_samechain(args)` and `native_fill_samechain(args)` — the same two deliveries, driven by a `CompactSamechainOrder` instead of a `MandateOutput`. Both must be called as top-level instructions (`NotTopLevel`), both record the signer as the solver rather than reading a `filler_data` field, and both create only a `FillRecord` — no `LocalAttestation`. Use them only after a matching `finalise_with_prefill` in the same transaction.

```rust theme={"system"}
pub struct FillSamechainArgs {
    pub order_id: [u8; 32],
    pub order: CompactSamechainOrder,
}

pub struct CompactSamechainOrder {
    pub nonce: u128,
    pub expires: u32,
    pub fill_deadline: u32,
    pub input_amount: u64,
    pub output_amount: u64,
    pub context: CompactSamechainContext,  // None | ExclusiveFor | ExclusiveForSelf
}
```

Everything else is rebuilt from accounts the instruction already authenticates: the chain id from the settler state, the recipient and output mint from the fill accounts (32 zero bytes for the native fill), and the solver from the signer. The rebuilt output always names the output-settler state PDA as both `oracle` and `settler` and carries no `callback_data`. `ExclusiveFor` uses the `exclusive_for` key it carries; only `ExclusiveForSelf` expands into the signer's own key. `None` expands to an empty context, so an order whose context is an explicit `0x00` byte cannot be filled this way.

`filler_data` is the 32-byte solver id recorded for the origin-chain claim; it is not derived from or compared with the Solana `filler` signer. Choose an identity that can authorize finalization on the input chain. In particular, when the input is on Solana, it must be the raw public-key bytes of the Solana signer that will call `finalise`.

The instruction accounts and failure conditions are defined by the program IDL and the validation rules above.

> [!NOTE]
>
> `output_settler_simple` requires `callback_data` to be empty. There is no post-fill callback execution on Solana.

### Fill records

Every fill instruction creates a `FillRecord` PDA — the double-fill guard: its existence blocks a second fill of the same output. The instructions name this account `fill_record`; `attest_not_filled` names the same address `fill_id`. Its seeds are `[order_id, mandate_output_hash]` — with no string prefix, unlike most PDAs in the system.

```rust theme={"system"}
/// PDA([order_id, mandate_output_hash]) under output_settler_simple.
pub struct FillRecord {
    pub rent_refund: Pubkey,  // the filler that paid the rent
    pub close_after: u64,     // fill_deadline + 172,800 seconds (48 hours)
}
```

`mandate_output_hash` is the keccak-256 of the output's packed encoding, not its Borsh encoding:

```text theme={"system"}
oracle[32] || settler[32] || chain_id[32] || token[32] || amount[32]
|| recipient[32] || callback_length[u16 BE] || callback_data
|| context_length[u16 BE] || context
```

`chain_id` and `amount` are their 32-byte big-endian values. The `order_id` seed is the origin chain's order identifier, byte for byte: the Borsh hash from [Order structure](#order-structure) for a Solana-origin order, or the origin chain's own order ID otherwise. A fill recorded under any other value cannot settle the order.

Once `close_after` has passed (strictly), anyone can call `close_fill_record(order_id, output_hash)` to close the record and return its rent to the recorded filler; the caller cannot redirect it. The timing is chosen so a record cannot be closed while a not-filled proof could still be created for that output. Closing emits `FillRecordClosedEvent`. Closing too early fails with `FillRecordNotClosable`.

> [!WARNING]
>
> The fill instructions take `fill_deadline` as an argument and only check it against the clock (`fill_deadline >= now`); they cannot compare it with the signed order. Always pass the order's exact `fill_deadline`: `close_after`, and with it the guarantee that no not-filled proof can be created while your record might be closed, is derived from the value you pass. A fill that executes after the order's signed `fill_deadline` — for example a second payment after the record was closed, made by passing a later deadline — delivers the tokens but can never settle the order, because `finalise` rejects fills timestamped after the signed `fill_deadline` (`FilledTooLate`).

> [!WARNING]
>
> Do not create orders with repeated outputs. The settler verifies, then fills: two identical outputs collapse into a single payment — the second fill is blocked by the first one's `FillRecord`, yet the order can still finalize. Make outputs distinct, for example with different amounts or recipients.

### Not-filled attestations

After an output's `fill_deadline` passes, anyone can call `attest_not_filled` to attest that the output was never filled. It requires the output's `FillRecord` PDA to be uninitialized — an output cannot be attested as not filled while its fill record exists — and creates a not-filled `LocalAttestation`, which unlocks the early refund path (`refund_on_non_fill`) on the input chain.

The attestation can only be created within a 48-hour window:

```text theme={"system"}
fill_deadline < now <= fill_deadline + 172,800
```

Calling it too early fails with `FillDeadlineNotPassed`; calling it after the window fails with `NotFilledWindowElapsed`. The window limits when the proof can be *created*, not how long it stays valid: a proof created in time can still be used for `refund_on_non_fill` later. If nobody creates one in time, the order can still be refunded through `refund` once it expires.

A non-fill refund cannot be undone: `finalise` rejects any fill timestamped after the order's signed `fill_deadline` (`FilledTooLate`), and the refund closes the order.

## Attestations

On EVM chains, proof that an output was filled lives in contract storage behind validation interfaces. Solana takes a different route: proofs are accounts. When a fill is validated, the protocol creates a PDA whose address is derived from the fill's content — so an account at the expected address is the proof. For a `LocalAttestation`, the account must also not be marked consumed (see [Consuming local attestations](#consuming-local-attestations)).

> [!NOTE]
>
> An attestation account is not a signature. It is a deterministic PDA that is created only when a specific fill — or proof of a fill — has been validated according to the protocol rules.

Two primitives cover the whole system:

* A **`LocalAttestation`** is created on Solana when an output is filled there with `fill` or `native_fill` (the compact fills do not create one). It proves the fill on the chain where it happened.
* An **`Attestation`** is created on Solana after an oracle verifies a remote proof or message. It represents a fill that happened on another chain.

Both account types, and the helpers that validate their PDAs, live in the `intents_protocol` program (IDL: [intents\_protocol.json](https://docs.li.fi/idl/intents_protocol.json)).

| Primitive | Created by | Used by | PDA seeds (in order) |
| - | - | - | - |
| `LocalAttestation` | Output settler, via CPI | Output-side oracle validation, same-chain finalization | `b"local_attestation"`, `attestator`, `consumer`, `data_hash` |
| `Attestation` | Input-side oracle, via CPI | Cross-chain finalization | `b"attestation"`, `attestator`, `chain_id_le`, `source`, `application`, `payload_hash` |

Pass only the seeds listed above to `PublicKey.findProgramAddressSync(seeds, programId)`. It finds and returns the bump automatically; do not append a bump to that input seed list. The returned bump is used separately when recreating or validating the PDA.

### What is being attested

An attestation commits to a payload describing what happened to one output. Two payload kinds exist, distinguished by a 4-byte magic prefix:

* A **`FillDescription`** says an output was filled. Its magic is the first 4 bytes of `keccak("OIF.Fill")` (`0xd1252dff`), and its layout is `MAGIC || solver (32B) || order_id (32B) || timestamp (u32 BE) || common_payload`.
* A **`NotFilledDescription`** says an output was not filled before its deadline. Its magic is the first 4 bytes of `keccak("OIF.NotFilled")` (`0x830c1e1c`), and its layout is `MAGIC || order_id (32B) || fill_deadline (u32 BE) || common_payload`.

`common_payload` encodes the output itself: `TOKEN (32B) || AMOUNT (32B) || RECIPIENT (32B) || CALLBACK_LENGTH (u16 BE) + CALLBACK_DATA || CONTEXT_LENGTH (u16 BE) + CONTEXT`. This packed, big-endian encoding is separate from the Borsh encoding used for instruction arguments and accounts.

A fill description is hashed two ways, and the difference matters when you debug a failed finalization:

* **`data_hash`**, used by `LocalAttestation`, is the keccak of the description **without** the timestamp. The timestamp is stored in the account and validated separately.
* **`payload_hash`**, used by `Attestation`, is the keccak of the **full** description, timestamp included.

For a not-filled description, both hashes cover the full payload, and the stored timestamp is fixed to the output's `fill_deadline`.

> [!NOTE]
>
> Length prefixes are `u16`, so `callback_data` and `context` are each capped at 65,535 bytes. The encoding is byte-compatible with the EVM implementation's `MandateOutputEncodingLib`, locked by cross-implementation test vectors.

### LocalAttestation: fills on Solana

When the output settler completes a fill, it computes the `data_hash` and creates the `LocalAttestation` by CPI into `intents_protocol`. The account's address is derived from `b"local_attestation"`, the `attestator` (the output settler PDA that signed the CPI), the `consumer`, the `data_hash`, and the bump returned by PDA derivation. The `consumer` is a 32-byte identifier for whoever is expected to consume the proof on the output chain — typically the output-side oracle program. The seed format is enforced on-chain by `validate_local_attestation(...)` in `intents_protocol`.

The timestamp comes from the Solana clock at fill time, converted to `u32` with a checked conversion (an out-of-range clock fails with `InvalidTimestamp`).

The account stores:

```rust theme={"system"}
/// 46 bytes including the 8-byte discriminator.
pub struct LocalAttestation {
    timestamp: u32,          // private; read it only after validating the PDA
    pub bump: u8,
    pub rent_refund: Pubkey, // whoever paid for the account — the filler, or the attest_not_filled caller
    pub consumed: bool,      // false at creation
}
```

### Consuming local attestations

A `LocalAttestation` is single-use for its committed consumer. When the consumer uses the proof irreversibly, it consumes the attestation through `intents_protocol::consume_local_attestation`, which validates the PDA, stored timestamp, and stored `rent_refund`, then closes the account and returns its rent to `rent_refund`. The `authority` signing the consume must be the consumer itself or its program's `["local_consumer"]` PDA.

| Consumer | Consumes in | Rent-refund slot |
| - | - | - |
| Polymer program (`output.oracle` for a cross-chain Solana output) | `submit_consume` | Interleaved `[attestation, rent_refund]` pairs |
| Wormhole oracle state PDA (its VAA emitter), which signs directly | `submit` | Interleaved `[attestation, rent_refund]` pairs |
| `input_settler_escrow` program (only if `output.oracle` is set to it) | `finalise`, `refund_on_non_fill` | Trailing slots — see [Settlement](#settlement) |

Anything else — including a same-chain attestation under the canonical encoding, whose consumer is the output settler — is validated but never consumed, and its rent stays locked.

If the stored `rent_refund` is an executable account, consumption still succeeds: the account is kept, marked `consumed = true`, and `LocalAttestationRentDeferredEvent` is emitted. `reclaim_local_attestation_rent` closes it later, and its destination must not be executable. Anyone can call it to return the rent to the recorded `rent_refund` once that account is no longer executable. To send the rent anywhere else, the recorded `rent_refund` must sign as `rent_refund_authority`. Only consumed attestations can be reclaimed (`LocalAttestationNotConsumed`).

A consumed attestation — closed, or kept with `consumed = true` — is no longer a valid proof, so it cannot be exported or used for settlement again.

### Attestation: proofs received from an oracle

When an oracle program verifies a remote proof — a Wormhole VAA, a Polymer proof — it registers each proven payload on Solana by CPI into `intents_protocol`. The address is derived from `b"attestation"`, the `attestator` (the input-side oracle named by `order.input_oracle`), `chain_id.to_le_bytes()` (little-endian `u128`), `source`, `application`, the `payload_hash`, and the bump returned by PDA derivation. Enforced on-chain by `validate_attestation(...)`.

The `source` and `application` seeds bind a proof to where it came from and what emitted it:

* `source` — the 32-byte identifier of the oracle or emitter on the output chain.
* `application` — the 32-byte identifier of the output settler, typically `MandateOutput.settler`.

At finalization, the input settler derives the expected PDA directly from the order's own fields: `chain_id` from `output.chain_id`, `source` from `output.oracle`, and `application` from `output.settler`.

> [!NOTE]
>
> This is why an order's `MandateOutput.oracle` and `MandateOutput.settler` must be the correct output-chain identifiers for the oracle system you use. If they are wrong, the proof lands at a PDA the finalization will never look at.

### The proof flow end to end

The two account types are the two halves of one relay, and they live on different chains. Steps 1 and 2 below run on the **output** chain (when an output is filled on Solana); steps 3 and 4 run on the **input** chain (when a Solana order awaits proof of a remote fill). For a single cross-chain order, Solana plays one of the two roles. A same-chain Solana order skips the relay entirely: finalization validates the fill's `LocalAttestation` directly (step 4).

**1. Fill creates a LocalAttestation (output chain)**

A solver fills an output on Solana through `output_settler_simple`. The settler transfers the assets, creates a `FillRecord` PDA against double fills, and creates a `LocalAttestation` committing to the fill's content and timestamp.

**2. The output-side oracle validates it (output chain)**

On the output chain, the oracle checks that the `LocalAttestation` exists at the correct PDA, is not consumed, and that its stored timestamp matches the payload: decode the `FillDescription`, recompute the `data_hash` without the timestamp, validate the PDA. Polymer's `submit` only validates; `submit_consume` and Wormhole's `submit` also consume the attestation.

**3. The input-side oracle creates an Attestation (input chain)**

On the input chain, the oracle verifies its own proof format — a Wormhole VAA or a Polymer proof — and creates one `Attestation` per proven `payload_hash`. The per-oracle details are in [Oracle systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md).

**4. Finalization checks one attestation per output (input chain)**

When the solver calls `finalise`, the input settler validates one attestation account per output, passed via `remaining_accounts` together with one rent-refund slot per output. A same-chain output expects a `LocalAttestation` whose timestamp matches the output's `SolveParams.timestamp`; a cross-chain output expects an `Attestation` at the PDA derived from `(input_oracle, chain_id, source, application, payload_hash)`.

For remote attestations, existence at the deterministic PDA is the entire proof: the oracle only creates the account after the underlying message has been verified.

### Non-fill attestations (early refunds)

The same two primitives carry proof that an output was **not** filled, which unlocks a refund before the order expires:

* On the output chain, `output_settler_simple::attest_not_filled` creates a `LocalAttestation` whose `data_hash` is the keccak of the full `NotFilledDescription` and whose stored timestamp is the output's `fill_deadline`.
* Oracles relay it exactly like a fill payload — the `NOT_FILLED_MAGIC` prefix selects the not-filled hashing rule on both sides.
* On the input chain, `refund_on_non_fill` validates the resulting `LocalAttestation` (same-chain) or `Attestation` (cross-chain) and refunds the escrow without waiting for `expires` — see [Refunds](#refunds).
* The not-filled `LocalAttestation` can only be created within 48 hours after `fill_deadline` — see [Not-filled attestations](#not-filled-attestations).

### Chain configuration

`intents_protocol` also stores the configuration the attestation system depends on:

* **`ChainId`** — a singleton PDA (seed `"chain_id"`) holding the deployment's canonical chain id. `set_chain_id(chain_id)` can be called exactly once, by any signer — deployments run it as the very first step, before anyone else can claim it. Settler programs copy the value at their own `initialize`.
* **`ChainMapping`** — a PDA (seeds `["chain_mapping", oracle, protocol_chain_id_le]`) that maps a messaging protocol's chain id to the canonical chain id. Each protocol has its own scheme: Polymer uses EVM chain ids for EVM chains but its own protocol id for Solana (`2`), and Wormhole uses its own `u16` ids. Each oracle's owner creates the mappings through the oracle's `set_chain_mapping(protocol_chain_id, chain_id)` instruction — owner-only, and each mapping can be set only once. Cross-chain attesting needs these mappings: if a proof arrives from a chain with no mapping, it cannot be registered.

### Events

`intents_protocol` emits an Anchor event for every state change, which indexers can subscribe to:

* `LocalAttestationCreatedEvent`
* `LocalAttestationConsumedEvent { attestator, consumer, data_hash, rent_refund }`
* `LocalAttestationRentDeferredEvent { local_attestation, rent_refund }` — consumed, but the rent is held because `rent_refund` is executable
* `LocalAttestationRentReclaimedEvent { local_attestation, rent_refund }`
* `AttestationCreatedEvent`
* `ChainIdConfiguredEvent`
* `ChainMapConfiguredEvent`

The other programs emit:

* `input_settler_escrow` — `OpenEvent`, `FinalisedEvent` (both finalization routes), `RefundedEvent`
* `output_settler_simple` — `OutputFilledEvent`, `OutputNotFilledEvent`, `FillRecordClosedEvent { order_id, output_hash, rent_refund }`
* `polymer` and `wormhole` — `OutputProvenEvent`

### Common pitfalls

* **Wrong 32-byte identifiers.** Every system has its own 32-byte representation: Solana `Pubkey::to_bytes()`, EVM addresses left-padded to 32 bytes, Wormhole emitter addresses. Mixing them up produces a PDA the validator will never find.
* **Each identity field fails at a different stage.** Check all three against the [identity matrix](#which-identity-goes-in-which-field) before filling:
  * If `MandateOutput.settler` is wrong, the fill is rejected. The order stays unfillable and refunds at expiry.
  * If `StandardOrder.input_oracle` is wrong, finalization derives the expected `Attestation` address using the wrong attestator, so it never finds the attestation the oracle actually created.
  * If `MandateOutput.oracle` is wrong, the fill still succeeds, because this field is not checked at fill time. The solver's assets leave immediately; proof submission later rejects the attestation, and the solver is not repaid.
* **Timestamp mismatches.** Same-chain validation requires `LocalAttestation.timestamp == SolveParams.timestamp`. If the solver passes a different timestamp than the one the output settler stored, finalization fails.
* **Missing rent-refund slots.** `finalise` and `refund_on_non_fill` expect a rent-refund slot for every attestation, even when it is ignored. Passing only the attestations fails with `WrongAttestationCount`.

## Same-chain intents

Solana-to-Solana intents skip remote proving entirely. Every same-chain order puts the output-settler state PDA in all three identity slots (`settler`, `oracle`, `inputOracle`) — see [Which identity goes in which field](#which-identity-goes-in-which-field). Such an order has two settlement routes:

* **Standard:** `fill` creates a `LocalAttestation`, and a later `finalise` validates it directly — no oracle program is involved. The attestation is not consumed, so its rent is not returned.
* **Input-first:** [input-first settlement](#input-first-settlement-finalise_with_prefill) pays the solver in the same transaction that delivers the output, so the solver can source the output from the user's input instead of from its own balance. The compact fill creates no attestation. The order must additionally have empty `callback_data` and an empty or exclusive-limit context. EVM reaches the same outcome through `finalise`'s callback; Solana has no callback and requires a specific later fill instruction instead.

![Same-chain Solana intent flow](https://mintcdn.com/lifi/OS19xH0Jt9xNmdgc/images/lifi-intent-solana-svm-same-chain.png?fit=max\&auto=format\&n=OS19xH0Jt9xNmdgc\&q=85\&s=e3cd07c19f174460b14c6f0bc12fe56c)

## Oracles

For cross-chain orders, oracle programs carry fill proofs from the output chain to the input chain and register them as `Attestation` accounts. Polymer and Wormhole deployments are listed in the [deployment overview](https://docs.li.fi/lifi-intents/architecture/overview.md#solana). Use the oracle configured for the order and route. How these implementations verify proofs is covered in [Oracle systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md), and the account model they write into is covered in [Attestations](#attestations) above.

## Extending the system

The programs are composable. Each concrete program wraps a shared base crate — `input_settler_base`, `output_settler_base`, and `oracle_base` — that carries the validation logic. You can build new settlers or oracles on the same bases, which keeps them compatible with the rest of the system.

For custom oracle authors, `oracle_base` provides the shared helpers. On the output side, `consume_and_encode_payloads` takes `(LocalAttestation, rent_refund)` pairs, validates and consumes each attestation with the consumption authority the oracle passes in (Polymer's `local_consumer` PDA, or Wormhole's own state PDA), and encodes the payloads into a single cross-chain message; `consume_payload` handles a single proof. On the input side, `decode_and_attest_payloads` decodes a message the oracle has already verified and creates one `Attestation` account per payload. Wormhole uses it; Polymer's `receive_attest` instead parses the proven EVM event itself.

## Program IDLs

The Anchor IDL for each program is the machine-readable interface reference: every instruction, account, type (with Borsh layout), event, and the program's own error codes. Errors shared across programs are in a separate catalog, [`errors.json`](https://docs.li.fi/idl/errors.json). Download them here and load them with Anchor's TypeScript client:

| Program | IDL |
| - | - |
| `input_settler_escrow` | [input\_settler\_escrow.json](https://docs.li.fi/idl/input_settler_escrow.json) |
| `output_settler_simple` | [output\_settler\_simple.json](https://docs.li.fi/idl/output_settler_simple.json) |
| `polymer` | [polymer.json](https://docs.li.fi/idl/polymer.json) |
| `wormhole` | [wormhole.json](https://docs.li.fi/idl/wormhole.json) |
| `intents_protocol` | [intents\_protocol.json](https://docs.li.fi/idl/intents_protocol.json) |

The four deployed-program IDLs embed the program IDs listed above. The Wormhole IDL is a source-interface reference: its embedded address differs from the [published mainnet deployment](https://docs.li.fi/lifi-intents/architecture/overview.md#solana). Deployment compatibility for that IDL has not been established here; do not substitute its address for the deployed program ID.

These are LI.FI's verified production IDLs. The [Solana IDL bundle](https://docs.li.fi/lifi-intents/architecture/solana-idl.md) page carries the rest of the bundle — the shared error catalog, the provenance record, the generated TypeScript types — and the rules an IDL cannot express: which PDAs to derive yourself, and the exact `remaining_accounts` layout of every instruction.

> [!WARNING]
>
> Some PDAs are seeded by values the program computes from instruction arguments, which an IDL cannot express, so the IDLs carry no seed metadata for them and Anchor will not resolve them automatically. Derive them client-side and pass them explicitly:
>
> * `order_context` = PDA(`["order_context", order_id]`) and `consumed_order` = PDA(`["consumed_order", order_id]`), where `order_id` is computed as in [Order structure](#order-structure) — used by `open`, `finalise`, `refund`, and `refund_on_non_fill`. Once `order_context` is supplied, Anchor derives the escrow token account from it.
> * `fill_record` = PDA(`[order_id, mandate_output_hash]`) on `output_settler_simple` — used by every fill instruction, and by `attest_not_filled` under the name `fill_id` (see [Fill records](#fill-records)).

## Next steps

> [card] [Oracle systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md)
>
> How Polymer verifies and relays the proofs

> [card] [Solana vs EVM](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md)
>
> What differs from the EVM implementation, and why


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.