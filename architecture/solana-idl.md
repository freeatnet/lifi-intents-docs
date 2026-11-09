> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Solana IDL bundle

> LI.FI's verified production IDLs for the Solana intent programs: downloads, the shared error catalog, generated TypeScript types, and the account rules an IDL cannot express.

The Solana programs publish a verified IDL bundle: one Anchor IDL per program, a shared error catalog, a provenance record, and generated TypeScript types. The IDLs are standard Anchor 0.31.1 output with corrected client metadata; the programs themselves are unchanged. This page lists the files and the rules you must apply by hand, because an IDL cannot express them.

For what each program does, see [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md).

## Downloads

| File | Contents |
| - | - |
| [input\_settler\_escrow.json](https://docs.li.fi/idl/input_settler_escrow.json) | Escrow input settler IDL |
| [output\_settler\_simple.json](https://docs.li.fi/idl/output_settler_simple.json) | Output settler IDL |
| [polymer.json](https://docs.li.fi/idl/polymer.json) | Polymer oracle IDL |
| [intents\_protocol.json](https://docs.li.fi/idl/intents_protocol.json) | Attestation and chain-configuration IDL |
| [wormhole.json](https://docs.li.fi/idl/wormhole.json) | Wormhole oracle IDL — source-interface reference; its embedded address differs from the published mainnet deployment |
| [errors.json](https://docs.li.fi/idl/errors.json) | Every first-party error catalog, and which catalogs each program can return |
| [provenance.json](https://docs.li.fi/idl/provenance.json) | Source revision, source and raw-IDL fingerprints, applied corrections, and artifact hashes |

The TypeScript types matching each IDL are under [TypeScript types](#typescript-types) below. Use the JSON files and the types together; running `anchor build` yourself produces raw IDLs without the corrections described here.

## Accounts you must derive yourself

Ten PDA expressions in the programs hash instruction arguments or call methods, which Anchor's automatic account resolver cannot execute. The published IDLs drop those `pda` hints, so Anchor will not fill these accounts in. Derive them client-side and pass them explicitly:

| Program | Instructions | Account | PDA seeds, in order |
| - | - | - | - |
| `input_settler_escrow` | `open`, `finalise`, `refund`, `refund_on_non_fill` | `order_context` | UTF-8 `order_context`, canonical order ID |
| `input_settler_escrow` | `open` | `consumed_order` | UTF-8 `consumed_order`, canonical order ID |
| `output_settler_simple` | `attest_not_filled` | `fill_id` | canonical order ID, canonical output hash |
| `output_settler_simple` | `fill`, `native_fill`, `fill_samechain`, `native_fill_samechain` | `fill_record` | canonical order ID, canonical output hash |

Derive each PDA under the program named in the first column. `fill_id` and `fill_record` are the same address under two names.

### Canonical order ID

Which value to use depends on where the order originated:

* **Input settler instructions** (`open`, `finalise`, `refund`, `refund_on_non_fill`) and the **compact same-chain fills** (`fill_samechain`, `native_fill_samechain`) use `keccak256(Borsh(StandardOrder))` — the order as signed for the input settler, and the fully expanded order for the compact path. Never hash JSON, the compact order bytes, or a SHA-256 digest instead.
* **Ordinary output fills** (`fill`, `native_fill`, `attest_not_filled`) take `order_id` as an argument and do **not** validate it. Pass the origin chain's order identifier byte for byte, exactly as that chain's input settler computed it. Only for an order opened on the Solana escrow is that the Borsh hash above; for an EVM or other origin, use that chain's own order ID. A fill recorded under any other value cannot settle the order.

### Canonical output hash

The output hash is keccak-256 over this packed encoding — not the Borsh encoding of `MandateOutput`:

```text theme={"system"}
oracle[32] || settler[32] || chain_id[32] || token[32] || amount[32]
|| recipient[32] || callback_length[u16 big-endian] || callback_data
|| context_length[u16 big-endian] || context
```

`chain_id` and `amount` are their 32-byte big-endian values, and each variable-length field must fit a `u16`.

For a compact same-chain fill, expand the output first, exactly as the program does: `oracle` and `settler` are the output-settler state PDA, `chain_id` comes from that state, `token` is the mint (32 zero bytes for native SOL), `amount` is `output_amount`, `recipient` is the recipient account, and `callback_data` is empty. `context` is empty for `None`, or `0xe0 || exclusive_for[32] || start_time[u32 big-endian]` — `ExclusiveForSelf` uses the filler's key as `exclusive_for`.

### Example: deriving the escrow accounts

```ts theme={"system"}
import { Program, type AnchorProvider } from "@coral-xyz/anchor";
import { PublicKey } from "@solana/web3.js";
import { keccak_256 } from "@noble/hashes/sha3.js";
import inputIdl from "./input_settler_escrow.json";
import type { InputSettlerEscrow } from "./types/input_settler_escrow";

// provider and order are supplied by your application. Bytes fields use Buffers;
// wide integer fields use BN, as required by the Anchor coder.
const program = new Program<InputSettlerEscrow>(
  inputIdl as unknown as InputSettlerEscrow,
  provider
);
const orderId = Buffer.from(
  keccak_256(program.coder.types.encode("standardOrder", order))
);
const [orderContext] = PublicKey.findProgramAddressSync(
  [Buffer.from("order_context"), orderId],
  program.programId
);
const [consumedOrder] = PublicKey.findProgramAddressSync(
  [Buffer.from("consumed_order"), orderId],
  program.programId
);
const instruction = await program.methods
  .open(order)
  .accountsStrict({
    ...allOtherOpenAccounts,
    orderContext,
    consumedOrder,
  })
  .instruction();
```

`allOtherOpenAccounts` must contain every other named account of `open`, because `accountsStrict` disables automatic resolution for the whole instruction. Use the camel-case account and argument names from the TypeScript types. Building an instruction does not check balances, ownership, hashes, or on-chain state. Enable TypeScript `strict` (or at least `strictNullChecks`) so Anchor's account types distinguish required accounts from optional ones.

## Compact same-chain fills

> [!WARNING]
>
> `fill_samechain` and `native_fill_samechain` must execute after a matching `finalise_with_prefill` in the **same transaction**. They do not create a `LocalAttestation`. Never submit them on their own or use them in place of ordinary fills.

Required instruction order, in one transaction:

```text theme={"system"}
finalise_with_prefill(order_id) → fill_samechain(args)
or
finalise_with_prefill(order_id) → native_fill_samechain(args)
```

`finalise_with_prefill` looks ahead for the matching compact fill and binds the supplied order ID to the expanded order before releasing the escrow. The compact fill handlers do not check for a preceding settlement instruction themselves, so a standalone compact fill pays the output but leaves the solver with nothing to claim against. The full workflow is in [Input-first Solana settlement](https://docs.li.fi/lifi-intents/for-solvers/settlement.md#input-first-solana-settlement).

## Remaining accounts

These accounts are appended with `.remainingAccounts(...)` after the named accounts; their number depends on the order, so they are not fixed IDL accounts. Below, N is the output or payload count, A is an attestation, and R is its stored rent-refund recipient. No remaining-account entry is a signer.

| Instruction | Required order | Permissions and signing |
| - | - | - |
| Escrow `finalise` | Exactly `[A0, …, A(N−1), R0, …, R(N−1)]` | A and R writable for local attestations committed to the escrow program. The named solver signs and matches the first output's solver. |
| Escrow `refund_on_non_fill` | Exactly `[A, R]` for the selected output | Both writable when the escrow program is the consumer. The named signer signs; the refund stays permissionless. |
| Polymer `submit` | Exactly `[A0, …, A(N−1)]` | Read-only; no refund slots and no instruction signer. N must be positive. |
| Polymer `submit_consume` | Exactly `[A0, R0, …, A(N−1), R(N−1)]` | All writable. N must be positive. The named submitter signs and must be the recorded filler of every fill payload; not-filled payloads are permissionless. |
| Wormhole `submit` | Exactly `[A0, R0, …, A(N−1), R(N−1)]` | All writable. N must be positive. The named submitter and a fresh post-message account sign; no filler-signature requirement. |
| Wormhole `receive` | Exactly `[A0, …, A(N−1)]`, in decoded VAA payload order | Writable destination attestation PDAs. The named signer funds their creation. No refund slots. |

For escrow settlement and refunds, remote attestations and local attestations committed to another consumer are not consumed. Their A accounts may be read-only, and their R slots are **required but ignored** — any existing read-only account, such as the system program, can fill an ignored slot. Never omit it. For a consumed attestation, pass the actual stored rent recipient. An executable recipient defers the rent reclaim but still needs its writable account meta. Duplicate addresses keep their positions; Solana merges the privileges of repeated keys when the transaction is compiled.

A two-payload `submit_consume`, for example:

```ts theme={"system"}
const remaining = [attestation0, refund0, attestation1, refund1].map(
  (pubkey) => ({ pubkey, isWritable: true, isSigner: false })
);
// Append with builder.remainingAccounts(remaining).
// A one-output finalise instead uses [attestation0, refund0].
```

## Events and errors

Each IDL lists the events its program emits. The Wormhole IDL includes `OutputProvenEvent`, emitted by the shared oracle code; its discriminator and field order match Polymer's: chain ID as `u128`, then three 32-byte fields (remote identifier, application, payload hash).

Each program IDL lists only its **local** errors. Shared errors — and the errors of programs it calls into — live in [`errors.json`](https://docs.li.fi/idl/errors.json), keyed by enum name, with the Rust source path, numeric code, name, and message of each. Its `programs` map lists, per program ID, the local catalog and the shared or CPI catalogs it can return.

| Catalog | Codes | Returned by |
| - | - | - |
| `CommonErrors` | 6000–6022 | Escrow, output settler, oracles |
| `IntentsProtocolErrors` | 6100–6110 | `intents_protocol`, and every program that calls it |
| `InputSettlerBaseErrors` | 6200–6201 | Escrow |
| `InputSettlerEscrowErrors` | 6300–6311 | Escrow (local) |
| `OutputSettlerBaseErrors` | 6400–6405 | Output settler |
| `OutputSettlerSimpleErrors` | 6500–6509 | Output settler (local) |
| `OracleBaseErrors` | 6600–6602 | Polymer, Wormhole |
| `OraclePolymerErrors` | 6700–6722 | Polymer (local) |
| `OracleWormholeErrors` | 6800–6803 | Wormhole (local) |

To decode a failure, first identify the program that actually failed from the invocation logs, then look the numeric code up in that program's local and shared catalogs. Prefer an explicit Anchor error name and message in the logs when one is present. Failures inside a CPI to `intents_protocol` use its catalog. Never label an unknown or external program's error with these codes, even when the numbers overlap; Anchor framework errors and external CPI errors need their own decoders. A catalog being associated with a program does not mean every variant is reachable from every instruction. Anchor's default error decoder does not read `errors.json` — load it yourself. [Debugging on Solana](https://docs.li.fi/lifi-intents/knowledge-database/solana-debugging.md) lists the errors you are most likely to meet.

## Provenance

The bundle describes the five production programs at source commit `776773a` of the Solana programs repository, built with Anchor 0.31.1. [`provenance.json`](https://docs.li.fi/idl/provenance.json) records the full commit, the source and raw-IDL fingerprints, every correction applied, and the hash of each published file. The corrections change only client metadata — removed `pda` hints and the added Wormhole event — never accounts, flags, arguments, or on-chain constraints. The published audit reports cover the reviewed contract revisions; the IDL tooling is a later integration change.

## TypeScript types

Camel-case TypeScript types matching each IDL, for use with Anchor's `Program<T>`. Save each one as `types/<program>.ts` next to the JSON files. Each type describes the corresponding JSON IDL exactly; if you regenerate IDLs yourself, regenerate the types from the same build.

> [accordion] types/input\_settler\_escrow\.ts
>
> ```ts input_settler_escrow.ts theme={"system"}
> /** Generated by yarn idl:generate. Runtime IDL: ../input_settler_escrow.json. */
> export type InputSettlerEscrow = {
>   "address": "LiFiRp8RM7nJUZyUYC9FPPpDr7sAy5XPfBN6ABzBgT7",
>   "metadata": {
>     "name": "inputSettlerEscrow",
>     "version": "0.0.0",
>     "spec": "0.1.0"
>   },
>   "instructions": [
>     {
>       "name": "finalise",
>       "docs": [
>         "Finalise a filled order and send the order's input funds to the order's solver.",
>         "`order` is the open order to be finalised.",
>         "`solve_params` are the solver parameters that filled each output (one per output). The first solver must be",
>         "the signer.",
>         "Remaining accounts: exactly 2*N accounts, where N is order.outputs.length: [attestation_0, ..., attestation_N-1, rent_refund_0, ..., rent_refund_N-1], in output order.",
>         "For local attestations committed to this input settler program, both the attestation and its stored rent-refund recipient must be writable. Executable recipients defer reclamation; keep both writable.",
>         "For remote or foreign-consumer attestations, the refund slot is required but ignored; those slots and attestations can be read-only. Remaining accounts require no additional signers. The named solver must sign and match the first output's solver."
>       ],
>       "discriminator": [
>         197,
>         63,
>         114,
>         249,
>         245,
>         72,
>         39,
>         132
>       ],
>       "accounts": [
>         {
>           "name": "solver",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "inputSettlerEscrow",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   105,
>                   110,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   101,
>                   115,
>                   99,
>                   114,
>                   111,
>                   119
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "user"
>         },
>         {
>           "name": "sponsor",
>           "docs": [
>             "via the `has_one = sponsor` constraint on `order_context`.",
>             "Receives both rent refunds unless executable, in which case they go to `solver`."
>           ],
>           "writable": true,
>           "relations": [
>             "orderContext"
>           ]
>         },
>         {
>           "name": "destination"
>         },
>         {
>           "name": "destinationTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "destination"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "orderContext",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "orderPdaTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "orderContext"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "mint",
>           "writable": true
>         },
>         {
>           "name": "localConsumer",
>           "docs": [
>             "no data). It signs the consumption of local attestations whose",
>             "committed consumer is this PROGRAM ID."
>           ],
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   108,
>                   111,
>                   99,
>                   97,
>                   108,
>                   95,
>                   99,
>                   111,
>                   110,
>                   115,
>                   117,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "tokenProgram"
>         },
>         {
>           "name": "associatedTokenProgram",
>           "address": "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "order",
>           "type": {
>             "defined": {
>               "name": "standardOrder"
>             }
>           }
>         },
>         {
>           "name": "solveParams",
>           "type": {
>             "vec": {
>               "defined": {
>                 "name": "solveParams"
>               }
>             }
>           }
>         }
>       ]
>     },
>     {
>       "name": "finaliseWithPrefill",
>       "docs": [
>         "Atomically release an open order's input before its compact same-chain fill executes.",
>         "The exact later fill is pinned through the Instructions sysvar."
>       ],
>       "discriminator": [
>         168,
>         65,
>         122,
>         15,
>         125,
>         60,
>         136,
>         108
>       ],
>       "accounts": [
>         {
>           "name": "solver",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "inputSettlerEscrow",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   105,
>                   110,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   101,
>                   115,
>                   99,
>                   114,
>                   111,
>                   119
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "user"
>         },
>         {
>           "name": "sponsor",
>           "docs": [
>             "executable, in which case they go to `solver`."
>           ],
>           "writable": true,
>           "relations": [
>             "orderContext"
>           ]
>         },
>         {
>           "name": "destination"
>         },
>         {
>           "name": "destinationTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "destination"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "orderContext",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   111,
>                   114,
>                   100,
>                   101,
>                   114,
>                   95,
>                   99,
>                   111,
>                   110,
>                   116,
>                   101,
>                   120,
>                   116
>                 ]
>               },
>               {
>                 "kind": "arg",
>                 "path": "orderId"
>               }
>             ]
>           }
>         },
>         {
>           "name": "orderPdaTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "orderContext"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "mint",
>           "writable": true
>         },
>         {
>           "name": "instructionsSysvar",
>           "address": "Sysvar1nstructions1111111111111111111111111"
>         },
>         {
>           "name": "tokenProgram"
>         },
>         {
>           "name": "associatedTokenProgram",
>           "address": "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "orderId",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         }
>       ]
>     },
>     {
>       "name": "initialize",
>       "docs": [
>         "Creates `input settler escrow` PDA.",
>         "`input settler escrow` PDA is used to store the chain identifier."
>       ],
>       "discriminator": [
>         175,
>         175,
>         109,
>         31,
>         13,
>         152,
>         155,
>         237
>       ],
>       "accounts": [
>         {
>           "name": "deployer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "chainId"
>         },
>         {
>           "name": "inputSettlerEscrow",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   105,
>                   110,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   101,
>                   115,
>                   99,
>                   114,
>                   111,
>                   119
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": []
>     },
>     {
>       "name": "open",
>       "docs": [
>         "Open the given `StandardOrder`. The order input funds will be pulled from the provided",
>         "user signer account, and will be escrowed until the order is finalised or expired."
>       ],
>       "discriminator": [
>         228,
>         220,
>         155,
>         71,
>         199,
>         189,
>         60,
>         45
>       ],
>       "accounts": [
>         {
>           "name": "sponsor",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "user",
>           "signer": true
>         },
>         {
>           "name": "inputSettlerEscrow",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   105,
>                   110,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   101,
>                   115,
>                   99,
>                   114,
>                   111,
>                   119
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "userTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "user"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "orderContext",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "consumedOrder",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "orderPdaTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "orderContext"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "mint"
>         },
>         {
>           "name": "tokenProgram"
>         },
>         {
>           "name": "associatedTokenProgram",
>           "address": "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "order",
>           "type": {
>             "defined": {
>               "name": "standardOrder"
>             }
>           }
>         }
>       ]
>     },
>     {
>       "name": "refund",
>       "docs": [
>         "Refund an order and send the order's input funds back to the user."
>       ],
>       "discriminator": [
>         2,
>         96,
>         183,
>         251,
>         63,
>         208,
>         46,
>         46
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "user"
>         },
>         {
>           "name": "sponsor",
>           "docs": [
>             "via the `has_one = sponsor` constraint on `order_context`.",
>             "Receives both rent refunds unless executable, in which case they go to `signer`."
>           ],
>           "writable": true,
>           "relations": [
>             "orderContext"
>           ]
>         },
>         {
>           "name": "inputSettlerEscrow",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   105,
>                   110,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   101,
>                   115,
>                   99,
>                   114,
>                   111,
>                   119
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "orderContext",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "orderPdaTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "orderContext"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "userTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "user"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "mint",
>           "writable": true
>         },
>         {
>           "name": "tokenProgram"
>         },
>         {
>           "name": "associatedTokenProgram",
>           "address": "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "order",
>           "type": {
>             "defined": {
>               "name": "standardOrder"
>             }
>           }
>         }
>       ]
>     },
>     {
>       "name": "refundOnNonFill",
>       "docs": [
>         "Refund an order as soon as one of its outputs is proven not filled before the fill deadline,",
>         "sending the order's input funds back to the user. Does not require `expires` to have passed.",
>         "`order` is the open order to be refunded.",
>         "`output_index` is the index of the output proven not filled.",
>         "Remaining accounts: exactly [attestation, rent_refund] for order.outputs[output_index].",
>         "For a local attestation committed to this input settler program, both accounts must be writable and rent_refund must be the stored recipient; executable recipients defer reclamation. Otherwise the refund slot is required but ignored and both accounts can be read-only.",
>         "Remaining accounts require no additional signers. The named signer must sign; the refund remains permissionless and goes to the order user."
>       ],
>       "discriminator": [
>         53,
>         164,
>         197,
>         243,
>         243,
>         43,
>         185,
>         108
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "user"
>         },
>         {
>           "name": "sponsor",
>           "docs": [
>             "via the `has_one = sponsor` constraint on `order_context`.",
>             "Receives both rent refunds unless executable, in which case they go to `signer`."
>           ],
>           "writable": true,
>           "relations": [
>             "orderContext"
>           ]
>         },
>         {
>           "name": "inputSettlerEscrow",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   105,
>                   110,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   101,
>                   115,
>                   99,
>                   114,
>                   111,
>                   119
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "orderContext",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "orderPdaTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "orderContext"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "userTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "user"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "mint",
>           "writable": true
>         },
>         {
>           "name": "localConsumer",
>           "docs": [
>             "no data). It signs the consumption of local attestations whose",
>             "committed consumer is this PROGRAM ID."
>           ],
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   108,
>                   111,
>                   99,
>                   97,
>                   108,
>                   95,
>                   99,
>                   111,
>                   110,
>                   115,
>                   117,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "tokenProgram"
>         },
>         {
>           "name": "associatedTokenProgram",
>           "address": "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "order",
>           "type": {
>             "defined": {
>               "name": "standardOrder"
>             }
>           }
>         },
>         {
>           "name": "outputIndex",
>           "type": "u8"
>         }
>       ]
>     }
>   ],
>   "accounts": [
>     {
>       "name": "chainId",
>       "discriminator": [
>         203,
>         146,
>         201,
>         161,
>         230,
>         71,
>         157,
>         218
>       ]
>     },
>     {
>       "name": "consumedOrder",
>       "discriminator": [
>         149,
>         74,
>         117,
>         35,
>         63,
>         65,
>         71,
>         140
>       ]
>     },
>     {
>       "name": "inputSettlerEscrowAccount",
>       "discriminator": [
>         85,
>         112,
>         162,
>         215,
>         22,
>         13,
>         16,
>         72
>       ]
>     },
>     {
>       "name": "orderContext",
>       "discriminator": [
>         240,
>         83,
>         159,
>         216,
>         26,
>         11,
>         225,
>         229
>       ]
>     }
>   ],
>   "events": [
>     {
>       "name": "finalisedEvent",
>       "discriminator": [
>         226,
>         131,
>         196,
>         252,
>         105,
>         222,
>         23,
>         210
>       ]
>     },
>     {
>       "name": "openEvent",
>       "discriminator": [
>         13,
>         191,
>         79,
>         145,
>         65,
>         183,
>         42,
>         90
>       ]
>     },
>     {
>       "name": "refundedEvent",
>       "discriminator": [
>         220,
>         3,
>         153,
>         244,
>         133,
>         189,
>         73,
>         119
>       ]
>     }
>   ],
>   "errors": [
>     {
>       "code": 6300,
>       "name": "solversLengthMismatch",
>       "msg": "Solvers length mismatch."
>     },
>     {
>       "code": 6301,
>       "name": "firstSolverNotInSolvers",
>       "msg": "First solver not in solvers."
>     },
>     {
>       "code": 6302,
>       "name": "finaliseTxTooLarge",
>       "msg": "Finalise transaction would exceed Solana's max packet size."
>     },
>     {
>       "code": 6303,
>       "name": "outputIndexOutOfBounds",
>       "msg": "Output index is out of bounds."
>     },
>     {
>       "code": 6304,
>       "name": "wrongAttestationCount",
>       "msg": "Exactly one attestation account is required."
>     },
>     {
>       "code": 6305,
>       "name": "notTopLevel",
>       "msg": "Input-first finalisation must be invoked as a top-level instruction."
>     },
>     {
>       "code": 6306,
>       "name": "missingSamechainFillInstruction",
>       "msg": "A matching future compact same-chain fill instruction is required."
>     },
>     {
>       "code": 6307,
>       "name": "ambiguousSamechainFillInstruction",
>       "msg": "More than one matching compact same-chain fill instruction was found."
>     },
>     {
>       "code": 6308,
>       "name": "invalidSamechainFillInstruction",
>       "msg": "The compact same-chain fill instruction is malformed or has invalid accounts."
>     },
>     {
>       "code": 6309,
>       "name": "invalidSamechainOrder",
>       "msg": "The compact same-chain order does not hash to the supplied order id."
>     },
>     {
>       "code": 6310,
>       "name": "invalidSamechainOrderContext",
>       "msg": "The order context does not match the compact same-chain order."
>     },
>     {
>       "code": 6311,
>       "name": "executableRentRefundAccount",
>       "msg": "The selected rent refund recipient must not be executable."
>     }
>   ],
>   "types": [
>     {
>       "name": "chainId",
>       "docs": [
>         "Type used to store the chain id of the chain."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "consumedOrder",
>       "docs": [
>         "This account should be used to track if an order has already been opened by a user.",
>         "It should be unique to each order for each user.",
>         "This means a user should not be allowed to open the same order twice.",
>         "This account should always live on chain and not be closed."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": []
>       }
>     },
>     {
>       "name": "finalisedEvent",
>       "docs": [
>         "Emitted by both `finalise` and `finalise_with_prefill`, mirroring the EVM",
>         "`Finalised(orderId, solver, destination)` event. Settled amounts are intentionally not",
>         "emitted."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "settler",
>             "type": "pubkey"
>           },
>           {
>             "name": "orderId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "solver",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "destination",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "inputSettlerEscrowAccount",
>       "docs": [
>         "Represents the input settler implementation type.",
>         "Implements the `InputSettlerBaseTrait` trait."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "mandateInput",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "token",
>             "type": "pubkey"
>           },
>           {
>             "name": "amount",
>             "type": "u64"
>           }
>         ]
>       }
>     },
>     {
>       "name": "mandateOutput",
>       "docs": [
>         "Defines the output state and where and how it should be filled.",
>         "DEV NOTES:",
>         "`oracle`: the implementation responsible for collecting the proof from settler on output chain",
>         "`settler`: the implementation on the output chain responsible for settling the output payment.",
>         "`callback_data`: the data to be passed to the callback function. MUST be empty if no callback is needed.",
>         "`context`: The output context for the output settlement, encoding order types or other information."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "oracle",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "settler",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "chainId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "token",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "amount",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "recipient",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "callbackData",
>             "type": "bytes"
>           },
>           {
>             "name": "context",
>             "type": "bytes"
>           }
>         ]
>       }
>     },
>     {
>       "name": "openEvent",
>       "docs": [
>         "Events must be defined directly in the program crate: Anchor's IDL",
>         "generator only collects `#[event]` structs from the crate that owns the",
>         "`#[program]` module."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "orderId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "order",
>             "type": {
>               "defined": {
>                 "name": "standardOrder"
>               }
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "orderContext",
>       "docs": [
>         "The context to opened as an account after opening an order.",
>         "`input_token` is the token account of the input \"Mint\".",
>         "`user` is the user who opened the order.",
>         "`sponsor` is the account that paid for the tx and the rent for this account.",
>         "Both escrow rent refunds go here on settlement unless it is executable, in which case",
>         "they go to the solver (finalisation) or signer (refund).",
>         "It can be the user itself or a third party that paid on the user's behalf.",
>         "`bump` is the bump of this account to be reused during finalisation."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "inputToken",
>             "type": "pubkey"
>           },
>           {
>             "name": "user",
>             "type": "pubkey"
>           },
>           {
>             "name": "sponsor",
>             "type": "pubkey"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "refundedEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "orderId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "solveParams",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "solver",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "timestamp",
>             "type": "u32"
>           }
>         ]
>       }
>     },
>     {
>       "name": "standardOrder",
>       "docs": [
>         "Defines the standard order structure.",
>         "DEV NOTES:",
>         "`expires`: the timestamp the order should be filled before. IMPORTANT for solvers to check.",
>         "`fill_deadline`: the timestamp for the funds to be reached to `user` on the output chain. IMPORTANT for solvers to check.",
>         "`input_oracle`: the oracle responsible for collecting the proof from the output chain."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "user",
>             "type": "pubkey"
>           },
>           {
>             "name": "nonce",
>             "type": "u128"
>           },
>           {
>             "name": "originChainId",
>             "type": "u128"
>           },
>           {
>             "name": "expires",
>             "type": "u32"
>           },
>           {
>             "name": "fillDeadline",
>             "type": "u32"
>           },
>           {
>             "name": "inputOracle",
>             "type": "pubkey"
>           },
>           {
>             "name": "input",
>             "type": {
>               "defined": {
>                 "name": "mandateInput"
>               }
>             }
>           },
>           {
>             "name": "outputs",
>             "type": {
>               "vec": {
>                 "defined": {
>                   "name": "mandateOutput"
>                 }
>               }
>             }
>           }
>         ]
>       }
>     }
>   ]
> };
> ```

> [accordion] types/output\_settler\_simple.ts
>
> ```ts output_settler_simple.ts theme={"system"}
> /** Generated by yarn idl:generate. Runtime IDL: ../output_settler_simple.json. */
> export type OutputSettlerSimple = {
>   "address": "LiFiEDFjz5x1jJe9gSXNDHQW4dWt4yLXdp2VN4EiQUt",
>   "metadata": {
>     "name": "outputSettlerSimple",
>     "version": "0.0.0",
>     "spec": "0.1.0"
>   },
>   "instructions": [
>     {
>       "name": "attestNotFilled",
>       "docs": [
>         "Permissionlessly attest that an order output was NOT filled before its fill deadline.",
>         "`order_id` is the input chain order identifier. Is not validated.",
>         "`output` is the output mandate that was not filled. Must target this settler on this chain.",
>         "`fill_deadline` is the fill deadline of the order; the current time must strictly exceed it."
>       ],
>       "discriminator": [
>         93,
>         101,
>         204,
>         135,
>         226,
>         64,
>         5,
>         137
>       ],
>       "accounts": [
>         {
>           "name": "payer",
>           "docs": [
>             "Permissionless payer funding the created `LocalAttestation` account."
>           ],
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "outputSettlerSimple",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   111,
>                   117,
>                   116,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   115,
>                   105,
>                   109,
>                   112,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "fillId",
>           "docs": [
>             "The `FillId` PDA for this output. It is created (`init`) during a fill, so it must be",
>             "un-initialised (system-owned, empty) here to prove no fill exists. The seed derivation is",
>             "owned by `output_settler_simple` (same scheme as `fill.rs`). Not `init`'d — only its address",
>             "is asserted so the account existence check inside the base trait is meaningful.",
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "localAttestation",
>           "writable": true
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "orderId",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "mandateOutput",
>           "type": {
>             "defined": {
>               "name": "mandateOutput"
>             }
>           }
>         },
>         {
>           "name": "fillDeadline",
>           "type": "u32"
>         }
>       ]
>     },
>     {
>       "name": "closeFillRecord",
>       "docs": [
>         "Permissionlessly close a `FillRecord` strictly after its stamped",
>         "`close_after`, returning the rent to the filler recorded at fill time",
>         "(never the caller).",
>         "`order_id` and `output_hash` address the record; no order data is read."
>       ],
>       "discriminator": [
>         60,
>         96,
>         240,
>         102,
>         51,
>         185,
>         23,
>         34
>       ],
>       "accounts": [
>         {
>           "name": "payer",
>           "docs": [
>             "Permissionless caller; pays only the transaction fee and receives",
>             "nothing."
>           ],
>           "signer": true
>         },
>         {
>           "name": "fillRecord",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "arg",
>                 "path": "orderId"
>               },
>               {
>                 "kind": "arg",
>                 "path": "outputHash"
>               }
>             ]
>           }
>         },
>         {
>           "name": "rentRefund",
>           "docs": [
>             "receives the account's lamports."
>           ],
>           "writable": true
>         }
>       ],
>       "args": [
>         {
>           "name": "orderId",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "outputHash",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         }
>       ]
>     },
>     {
>       "name": "fill",
>       "docs": [
>         "Fill an order output.",
>         "`order_id` is the input chain order identifier. Is not validated.",
>         "`output` is the output mandate to be filled.",
>         "`fill_deadline` is the maximum time in seconds the fill may be executed at.",
>         "`filler_data` contains the solver identifier sent to the input chain."
>       ],
>       "discriminator": [
>         168,
>         96,
>         183,
>         163,
>         92,
>         10,
>         40,
>         160
>       ],
>       "accounts": [
>         {
>           "name": "filler",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "recipient",
>           "writable": true
>         },
>         {
>           "name": "outputSettlerSimple",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   111,
>                   117,
>                   116,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   115,
>                   105,
>                   109,
>                   112,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "fillerTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "filler"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "recipientTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "recipient"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "mint"
>         },
>         {
>           "name": "fillRecord",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "localAttestation",
>           "writable": true
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "tokenProgram"
>         },
>         {
>           "name": "associatedTokenProgram",
>           "address": "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "orderId",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "mandateOutput",
>           "type": {
>             "defined": {
>               "name": "mandateOutput"
>             }
>           }
>         },
>         {
>           "name": "fillDeadline",
>           "type": "u64"
>         },
>         {
>           "name": "fillerData",
>           "type": "bytes"
>         }
>       ],
>       "returns": "pubkey"
>     },
>     {
>       "name": "fillSamechain",
>       "docs": [
>         "Fill the single token output of a compact same-chain order.",
>         "Warning: Compact same-chain fills must execute after a matching finalise_with_prefill in the same transaction. They do not create a LocalAttestation. Never submit them independently or substitute them for ordinary attested fills.",
>         "finalise_with_prefill checks for a matching future compact fill and binds the supplied order ID to the expanded order before releasing the input escrow. The compact fill handlers do not enforce a preceding settlement instruction themselves. A standalone compact fill can pay the output while producing an unclaimable fill, without the proof needed for ordinary later finalisation.",
>         "Required instruction order, all in one transaction: finalise_with_prefill(orderId) -> fill_samechain(args)."
>       ],
>       "discriminator": [
>         41,
>         219,
>         215,
>         115,
>         27,
>         174,
>         177,
>         210
>       ],
>       "accounts": [
>         {
>           "name": "filler",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "recipient",
>           "docs": [
>             "attestation PDAs commit to it and the input-side order-id validation pins it."
>           ],
>           "writable": true
>         },
>         {
>           "name": "outputSettlerSimple",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   111,
>                   117,
>                   116,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   115,
>                   105,
>                   109,
>                   112,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "fillerTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "filler"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "recipientTokenAccount",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "account",
>                 "path": "recipient"
>               },
>               {
>                 "kind": "account",
>                 "path": "tokenProgram"
>               },
>               {
>                 "kind": "account",
>                 "path": "mint"
>               }
>             ],
>             "program": {
>               "kind": "const",
>               "value": [
>                 140,
>                 151,
>                 37,
>                 143,
>                 78,
>                 36,
>                 137,
>                 241,
>                 187,
>                 61,
>                 16,
>                 41,
>                 20,
>                 142,
>                 13,
>                 131,
>                 11,
>                 90,
>                 19,
>                 153,
>                 218,
>                 255,
>                 16,
>                 132,
>                 4,
>                 142,
>                 123,
>                 216,
>                 219,
>                 233,
>                 248,
>                 89
>               ]
>             }
>           }
>         },
>         {
>           "name": "mint"
>         },
>         {
>           "name": "fillRecord",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "tokenProgram"
>         },
>         {
>           "name": "associatedTokenProgram",
>           "address": "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "args",
>           "type": {
>             "defined": {
>               "name": "fillSamechainArgs"
>             }
>           }
>         }
>       ],
>       "returns": "pubkey"
>     },
>     {
>       "name": "initialize",
>       "docs": [
>         "Creates `output settler simple` PDA.",
>         "`output settler simple` PDA is used to store the chain identifier."
>       ],
>       "discriminator": [
>         175,
>         175,
>         109,
>         31,
>         13,
>         152,
>         155,
>         237
>       ],
>       "accounts": [
>         {
>           "name": "deployer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "chainId"
>         },
>         {
>           "name": "outputSettlerSimple",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   111,
>                   117,
>                   116,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   115,
>                   105,
>                   109,
>                   112,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [],
>       "returns": "pubkey"
>     },
>     {
>       "name": "nativeFill",
>       "docs": [
>         "Fill an order output with native SOL.",
>         "`order_id` is the input chain order identifier. Is not validated.",
>         "`output` is the output mandate to be filled.",
>         "`fill_deadline` is the maximum time in seconds the fill may be executed at.",
>         "`filler_data` contains the solver identifier sent to the input chain."
>       ],
>       "discriminator": [
>         49,
>         10,
>         255,
>         151,
>         120,
>         148,
>         73,
>         30
>       ],
>       "accounts": [
>         {
>           "name": "filler",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "recipient",
>           "writable": true
>         },
>         {
>           "name": "outputSettlerSimple",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   111,
>                   117,
>                   116,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   115,
>                   105,
>                   109,
>                   112,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "fillRecord",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "localAttestation",
>           "writable": true
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "orderId",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "mandateOutput",
>           "type": {
>             "defined": {
>               "name": "mandateOutput"
>             }
>           }
>         },
>         {
>           "name": "fillDeadline",
>           "type": "u64"
>         },
>         {
>           "name": "fillerData",
>           "type": "bytes"
>         }
>       ],
>       "returns": "pubkey"
>     },
>     {
>       "name": "nativeFillSamechain",
>       "docs": [
>         "Fill the single native SOL output of a compact same-chain order.",
>         "Warning: Compact same-chain fills must execute after a matching finalise_with_prefill in the same transaction. They do not create a LocalAttestation. Never submit them independently or substitute them for ordinary attested fills.",
>         "finalise_with_prefill checks for a matching future compact fill and binds the supplied order ID to the expanded order before releasing the input escrow. The compact fill handlers do not enforce a preceding settlement instruction themselves. A standalone compact fill can pay the output while producing an unclaimable fill, without the proof needed for ordinary later finalisation.",
>         "Required instruction order, all in one transaction: finalise_with_prefill(orderId) -> native_fill_samechain(args)."
>       ],
>       "discriminator": [
>         246,
>         44,
>         97,
>         75,
>         180,
>         183,
>         72,
>         72
>       ],
>       "accounts": [
>         {
>           "name": "filler",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "recipient",
>           "docs": [
>             "attestation PDAs commit to it and the input-side order-id validation pins it."
>           ],
>           "writable": true
>         },
>         {
>           "name": "outputSettlerSimple",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   111,
>                   117,
>                   116,
>                   112,
>                   117,
>                   116,
>                   95,
>                   115,
>                   101,
>                   116,
>                   116,
>                   108,
>                   101,
>                   114,
>                   95,
>                   115,
>                   105,
>                   109,
>                   112,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "fillRecord",
>           "writable": true,
>           "docs": [
>             "Supply this PDA explicitly. Anchor cannot represent the existing hashed seed expression. Use the canonical order/output hash and original program seeds; see this bundle's README.md. Runtime constraints are unchanged."
>           ]
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "args",
>           "type": {
>             "defined": {
>               "name": "fillSamechainArgs"
>             }
>           }
>         }
>       ],
>       "returns": "pubkey"
>     }
>   ],
>   "accounts": [
>     {
>       "name": "chainId",
>       "discriminator": [
>         203,
>         146,
>         201,
>         161,
>         230,
>         71,
>         157,
>         218
>       ]
>     },
>     {
>       "name": "fillRecord",
>       "discriminator": [
>         27,
>         79,
>         245,
>         54,
>         39,
>         229,
>         168,
>         24
>       ]
>     },
>     {
>       "name": "outputSettlerSimpleAccount",
>       "discriminator": [
>         227,
>         35,
>         185,
>         166,
>         177,
>         53,
>         143,
>         129
>       ]
>     }
>   ],
>   "events": [
>     {
>       "name": "fillRecordClosedEvent",
>       "discriminator": [
>         70,
>         227,
>         88,
>         40,
>         245,
>         228,
>         143,
>         59
>       ]
>     },
>     {
>       "name": "outputFilledEvent",
>       "discriminator": [
>         43,
>         25,
>         133,
>         52,
>         107,
>         176,
>         40,
>         213
>       ]
>     },
>     {
>       "name": "outputNotFilledEvent",
>       "discriminator": [
>         162,
>         22,
>         108,
>         52,
>         116,
>         186,
>         115,
>         37
>       ]
>     }
>   ],
>   "errors": [
>     {
>       "code": 6500,
>       "name": "notImplemented",
>       "msg": "Order type not implemented."
>     },
>     {
>       "code": 6501,
>       "name": "amountTooLarge",
>       "msg": "Amount is too large."
>     },
>     {
>       "code": 6502,
>       "name": "arithmeticOverflow",
>       "msg": "Arithmetic overflow."
>     },
>     {
>       "code": 6503,
>       "name": "invalidFillerData",
>       "msg": "Invalid filler data."
>     },
>     {
>       "code": 6504,
>       "name": "invalidContextDataLength",
>       "msg": "Invalid context data length."
>     },
>     {
>       "code": 6505,
>       "name": "exclusiveTo",
>       "msg": "Order is exclusive to another solver or time passed."
>     },
>     {
>       "code": 6506,
>       "name": "zeroValue",
>       "msg": "Zero value is not allowed."
>     },
>     {
>       "code": 6507,
>       "name": "notTopLevel",
>       "msg": "Compact same-chain fills must be invoked as top-level instructions."
>     },
>     {
>       "code": 6508,
>       "name": "fillRecordNotClosable",
>       "msg": "Fill record is not closable yet."
>     },
>     {
>       "code": 6509,
>       "name": "invalidRentRefund",
>       "msg": "Rent refund account does not match the recorded rent refund."
>     }
>   ],
>   "types": [
>     {
>       "name": "chainId",
>       "docs": [
>         "Type used to store the chain id of the chain."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "compactSamechainContext",
>       "docs": [
>         "The only pricing contexts accepted by the compact input-first path.",
>         "",
>         "Variant order is wire ABI: do not reorder. Both exclusive variants expand to the canonical",
>         "`0xe0 || exclusive_for || start_time_be` context; `ExclusiveForSelf` is a transport",
>         "optimization that substitutes the current solver for `exclusive_for`."
>       ],
>       "type": {
>         "kind": "enum",
>         "variants": [
>           {
>             "name": "none"
>           },
>           {
>             "name": "exclusiveFor",
>             "fields": [
>               {
>                 "name": "exclusiveFor",
>                 "type": {
>                   "array": [
>                     "u8",
>                     32
>                   ]
>                 }
>               },
>               {
>                 "name": "startTime",
>                 "type": "u32"
>               }
>             ]
>           },
>           {
>             "name": "exclusiveForSelf",
>             "fields": [
>               {
>                 "name": "startTime",
>                 "type": "u32"
>               }
>             ]
>           }
>         ]
>       }
>     },
>     {
>       "name": "compactSamechainOrder",
>       "docs": [
>         "Compact, same-chain-only representation carried by `fill_samechain` instructions.",
>         "",
>         "Fields recoverable from the input and output instruction accounts are deliberately omitted.",
>         "Every protocol hash and PDA is derived from the canonical `StandardOrder` or `MandateOutput`",
>         "encoding reconstructed by expansion, never from the compact bytes themselves."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "nonce",
>             "type": "u128"
>           },
>           {
>             "name": "expires",
>             "type": "u32"
>           },
>           {
>             "name": "fillDeadline",
>             "type": "u32"
>           },
>           {
>             "name": "inputAmount",
>             "type": "u64"
>           },
>           {
>             "name": "outputAmount",
>             "type": "u64"
>           },
>           {
>             "name": "context",
>             "type": {
>               "defined": {
>                 "name": "compactSamechainContext"
>               }
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "fillRecord",
>       "docs": [
>         "Created during fill to ensure no double fill for the same output, and read",
>         "by `attest_not_filled` as the proof-of-fill via its *absence* check. Lives at",
>         "the PDA seeds `[order_id, mandate_output_hash]`.",
>         "",
>         "`rent_refund` is the filler that funded the account; `close_after` is",
>         "`fill_deadline + NOT_FILLED_WINDOW_SECONDS`, stamped identically in all four",
>         "fill paths and immutable afterwards. The permissionless `close_fill_record`",
>         "reclaims the rent to `rent_refund` strictly after `close_after`."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "rentRefund",
>             "type": "pubkey"
>           },
>           {
>             "name": "closeAfter",
>             "type": "u64"
>           }
>         ]
>       }
>     },
>     {
>       "name": "fillRecordClosedEvent",
>       "docs": [
>         "Emitted when a `FillRecord` past its `close_after` is closed and its rent",
>         "returned to the filler recorded at fill time."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "orderId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "outputHash",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "rentRefund",
>             "type": "pubkey"
>           }
>         ]
>       }
>     },
>     {
>       "name": "fillSamechainArgs",
>       "docs": [
>         "Shared instruction payload for token and native compact same-chain fills."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "orderId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "order",
>             "type": {
>               "defined": {
>                 "name": "compactSamechainOrder"
>               }
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "mandateOutput",
>       "docs": [
>         "Defines the output state and where and how it should be filled.",
>         "DEV NOTES:",
>         "`oracle`: the implementation responsible for collecting the proof from settler on output chain",
>         "`settler`: the implementation on the output chain responsible for settling the output payment.",
>         "`callback_data`: the data to be passed to the callback function. MUST be empty if no callback is needed.",
>         "`context`: The output context for the output settlement, encoding order types or other information."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "oracle",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "settler",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "chainId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "token",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "amount",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "recipient",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "callbackData",
>             "type": "bytes"
>           },
>           {
>             "name": "context",
>             "type": "bytes"
>           }
>         ]
>       }
>     },
>     {
>       "name": "outputFilledEvent",
>       "docs": [
>         "IDL-only mirrors of the events emitted by `output_settler_base`.",
>         "",
>         "The events are emitted from inside `output_settler_base` (see",
>         "`base.rs`), but Anchor's IDL generator only collects `#[event]` structs",
>         "defined in the program crate itself, so without these mirrors the events",
>         "are silently absent from the published IDL. Anchor identifies an event by",
>         "its name-derived discriminator (`sha256(\"event:<Name>\")[..8]`) and its",
>         "borsh field layout, so an identical copy here yields the exact IDL entry",
>         "for the event the base crate emits. The tests below pin the mirrors to the",
>         "base-crate originals byte-for-byte."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "settler",
>             "type": "pubkey"
>           },
>           {
>             "name": "orderId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "solver",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "timestamp",
>             "type": "u32"
>           },
>           {
>             "name": "output",
>             "type": {
>               "defined": {
>                 "name": "mandateOutput"
>               }
>             }
>           },
>           {
>             "name": "finalAmount",
>             "type": "u64"
>           }
>         ]
>       }
>     },
>     {
>       "name": "outputNotFilledEvent",
>       "docs": [
>         "Informational event emitted when an output is provably attested as not filled before its fill",
>         "deadline. Mirrors the EVM `OutputNotFilled(orderId, output, fillDeadline)` event."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "settler",
>             "type": "pubkey"
>           },
>           {
>             "name": "orderId",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "fillDeadline",
>             "type": "u32"
>           },
>           {
>             "name": "output",
>             "type": {
>               "defined": {
>                 "name": "mandateOutput"
>               }
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "outputSettlerSimpleAccount",
>       "docs": [
>         "Represents the output settler implementation type.",
>         "Implements the `OutputSettlerBaseTrait` trait."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     }
>   ]
> };
> ```

> [accordion] types/polymer.ts
>
> ```ts polymer.ts theme={"system"}
> /** Generated by yarn idl:generate. Runtime IDL: ../polymer.json. */
> export type Polymer = {
>   "address": "LiFiBtfyPT1DnTHTAeZ2rwr5RgMrThwA5kt7KGT5nBV",
>   "metadata": {
>     "name": "polymer",
>     "version": "0.0.0",
>     "spec": "0.1.0"
>   },
>   "docs": [
>     "There are two ways to receive a remote proof, both ending in",
>     "[`receive_attest`] which reads the validated event data from the prover's",
>     "`result_account` and registers the attestation:",
>     "",
>     "1. **Single step** — call [`receive_load_proof`] with the full proof. It",
>     "runs the prover's `create_accounts`, `load_proof` and `validate_event`",
>     "CPIs in one transaction, then `receive_attest` finalises it. Use this",
>     "when the proof fits within a single Solana transaction.",
>     "",
>     "2. **Three steps** — for proofs too large to fit in a single Solana",
>     "transaction, the load is chunked across multiple transactions:",
>     "1. [`create_polymer_accounts`] — create the prover's `cache`/`result` PDAs.",
>     "2. [`load_proof_chunk`] — append one chunk of the proof; call once per",
>     "chunk until the full proof is loaded (the prover extends the cache).",
>     "3. [`validate_event`] — validate the accumulated proof in the cache.",
>     "Then `receive_attest` finalises it as above."
>   ],
>   "instructions": [
>     {
>       "name": "createPolymerAccounts",
>       "docs": [
>         "Chunked receive step 1: create the polymer prover's `cache` and",
>         "`result` scratch PDAs. Use this path instead of `receive_load_proof`",
>         "when the proof is too large to load in a single transaction."
>       ],
>       "discriminator": [
>         238,
>         43,
>         4,
>         23,
>         70,
>         95,
>         159,
>         84
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "polymerProverProgram"
>         },
>         {
>           "name": "cacheAccount",
>           "writable": true
>         },
>         {
>           "name": "resultAccount",
>           "writable": true
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": []
>     },
>     {
>       "name": "initialize",
>       "docs": [
>         "Initializes the `oracle_polymer` account. Creates `oracle_polymer` PDA and sets the `polymer_prover_id`.",
>         "`oracle_polymer` PDA is used to store the chain identifier and the owner address."
>       ],
>       "discriminator": [
>         175,
>         175,
>         109,
>         31,
>         13,
>         152,
>         155,
>         237
>       ],
>       "accounts": [
>         {
>           "name": "deployer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "chainId"
>         },
>         {
>           "name": "oraclePolymer",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "polymerProverId",
>           "type": "pubkey"
>         }
>       ]
>     },
>     {
>       "name": "loadProofChunk",
>       "docs": [
>         "Chunked receive step 2: append a single chunk of the proof into the",
>         "prover's `cache` PDA. Call repeatedly until the full proof is loaded."
>       ],
>       "discriminator": [
>         252,
>         35,
>         79,
>         22,
>         76,
>         152,
>         132,
>         74
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "polymerProverProgram"
>         },
>         {
>           "name": "cacheAccount",
>           "writable": true
>         }
>       ],
>       "args": [
>         {
>           "name": "proofChunk",
>           "type": "bytes"
>         }
>       ]
>     },
>     {
>       "name": "receiveAttest",
>       "docs": [
>         "Step 2 of receive: read the validated event data left by",
>         "`receive_load_proof`, register the attestation, and close the polymer",
>         "prover's scratch PDAs."
>       ],
>       "discriminator": [
>         112,
>         136,
>         131,
>         35,
>         95,
>         206,
>         124,
>         161
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "polymerProverProgram"
>         },
>         {
>           "name": "chainMapping"
>         },
>         {
>           "name": "cacheAccount",
>           "writable": true
>         },
>         {
>           "name": "internal",
>           "writable": true
>         },
>         {
>           "name": "resultAccount",
>           "writable": true
>         },
>         {
>           "name": "attestation",
>           "writable": true
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": []
>     },
>     {
>       "name": "receiveLoadProof",
>       "docs": [
>         "Step 1 of receive: load and validate a single proof via the polymer",
>         "prover. Leaves the validated event data in `result_account` for",
>         "`receive_attest` to consume in a follow-up tx."
>       ],
>       "discriminator": [
>         220,
>         77,
>         15,
>         159,
>         191,
>         141,
>         155,
>         51
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "polymerProverProgram"
>         },
>         {
>           "name": "cacheAccount",
>           "writable": true
>         },
>         {
>           "name": "internal",
>           "writable": true
>         },
>         {
>           "name": "resultAccount",
>           "writable": true
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "proof",
>           "type": "bytes"
>         }
>       ]
>     },
>     {
>       "name": "setChainMapping",
>       "docs": [
>         "Maps polymer chain id to a configured custom chain id.",
>         "Can only be called by the owner of the `oracle_polymer` account.",
>         "Each mapping may only be set once."
>       ],
>       "discriminator": [
>         174,
>         145,
>         44,
>         171,
>         203,
>         101,
>         230,
>         130
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "chainMapping",
>           "writable": true
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "protocolChainId",
>           "type": "u128"
>         },
>         {
>           "name": "chainId",
>           "type": "u128"
>         }
>       ]
>     },
>     {
>       "name": "submit",
>       "docs": [
>         "Permissionlessly validates and logs proofs, retaining their local attestations",
>         "and rent for repeat submissions. Requires one read-only attestation per payload.",
>         "Remaining accounts: exactly one read-only local_attestation per fill_descriptions entry, in payload order. No remaining-account signers or refund slots. Payloads must not be empty. This instruction retains attestations for repeat submissions."
>       ],
>       "discriminator": [
>         88,
>         166,
>         102,
>         181,
>         162,
>         127,
>         170,
>         48
>       ],
>       "accounts": [
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         }
>       ],
>       "args": [
>         {
>           "name": "source",
>           "type": "pubkey"
>         },
>         {
>           "name": "fillDescriptions",
>           "type": {
>             "vec": "bytes"
>           }
>         }
>       ]
>     },
>     {
>       "name": "submitConsume",
>       "docs": [
>         "Validates, consumes and logs proofs, returning rent when possible.",
>         "Every fill requires its recorded filler to sign as `submitter`.",
>         "Not-filled proofs remain permissionless. Requires one writable",
>         "`(local_attestation, rent_refund)` pair per payload.",
>         "Remaining accounts: exactly [local_attestation_0, rent_refund_0, ..., local_attestation_N-1, rent_refund_N-1] in fill_descriptions order. Payloads must not be empty. Both accounts in every pair must be writable; no additional remaining-account signers.",
>         "Each rent_refund must be the attestation's stored recipient. Executable recipients defer reclamation; keep both writable. Every fill payload requires its recorded filler to sign as the named submitter; not-filled payloads remain permissionless."
>       ],
>       "discriminator": [
>         19,
>         162,
>         196,
>         241,
>         27,
>         133,
>         47,
>         49
>       ],
>       "accounts": [
>         {
>           "name": "submitter",
>           "docs": [
>             "Must be the recorded rent-refund recipient (the filler) for every fill",
>             "payload. Not-filled payloads may be submitted by any signer."
>           ],
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "localConsumer",
>           "docs": [
>             "no data). The committed consumer of Polymer-routed attestations is this",
>             "PROGRAM ID, which cannot sign; the intents protocol accepts this PDA as",
>             "the program's consumption authority instead."
>           ],
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   108,
>                   111,
>                   99,
>                   97,
>                   108,
>                   95,
>                   99,
>                   111,
>                   110,
>                   115,
>                   117,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         }
>       ],
>       "args": [
>         {
>           "name": "source",
>           "type": "pubkey"
>         },
>         {
>           "name": "fillDescriptions",
>           "type": {
>             "vec": "bytes"
>           }
>         }
>       ]
>     },
>     {
>       "name": "validateEvent",
>       "docs": [
>         "Chunked receive step 3: validate the proof accumulated in the `cache`",
>         "PDA, leaving the validated event data in `result_account` for",
>         "`receive_attest` to consume in a follow-up tx."
>       ],
>       "discriminator": [
>         66,
>         249,
>         207,
>         221,
>         30,
>         87,
>         27,
>         129
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oraclePolymer",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   112,
>                   111,
>                   108,
>                   121,
>                   109,
>                   101,
>                   114
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "polymerProverProgram"
>         },
>         {
>           "name": "cacheAccount",
>           "writable": true
>         },
>         {
>           "name": "internal",
>           "writable": true
>         },
>         {
>           "name": "resultAccount",
>           "writable": true
>         }
>       ],
>       "args": []
>     }
>   ],
>   "accounts": [
>     {
>       "name": "chainId",
>       "discriminator": [
>         203,
>         146,
>         201,
>         161,
>         230,
>         71,
>         157,
>         218
>       ]
>     },
>     {
>       "name": "chainMapping",
>       "discriminator": [
>         81,
>         78,
>         77,
>         209,
>         44,
>         206,
>         85,
>         169
>       ]
>     },
>     {
>       "name": "oraclePolymer",
>       "discriminator": [
>         30,
>         219,
>         153,
>         176,
>         126,
>         163,
>         80,
>         138
>       ]
>     }
>   ],
>   "events": [
>     {
>       "name": "outputProvenEvent",
>       "discriminator": [
>         85,
>         190,
>         185,
>         8,
>         90,
>         48,
>         3,
>         214
>       ]
>     }
>   ],
>   "errors": [
>     {
>       "code": 6700,
>       "name": "invalidChainId",
>       "msg": "Invalid chain id"
>     },
>     {
>       "code": 6701,
>       "name": "invalidProof",
>       "msg": "Invalid proof"
>     },
>     {
>       "code": 6702,
>       "name": "invalidOutputChainId",
>       "msg": "Invalid output chain id"
>     },
>     {
>       "code": 6703,
>       "name": "unauthorized",
>       "msg": "Unauthorized mapping creation"
>     },
>     {
>       "code": 6704,
>       "name": "couldNotDecodeSolver",
>       "msg": "Could not decode solver"
>     },
>     {
>       "code": 6705,
>       "name": "couldNotDecodeTimestamp",
>       "msg": "Could not decode timestamp"
>     },
>     {
>       "code": 6706,
>       "name": "couldNotDecodeMandateOutputFieldsTuple",
>       "msg": "Could not decode mandate output fields tuple"
>     },
>     {
>       "code": 6707,
>       "name": "couldNotDecodeOracle",
>       "msg": "Could not decode oracle"
>     },
>     {
>       "code": 6708,
>       "name": "couldNotDecodeSettler",
>       "msg": "Could not decode settler"
>     },
>     {
>       "code": 6709,
>       "name": "couldNotDecodeChainId",
>       "msg": "Could not decode chain id"
>     },
>     {
>       "code": 6710,
>       "name": "couldNotDecodeToken",
>       "msg": "Could not decode token"
>     },
>     {
>       "code": 6711,
>       "name": "couldNotDecodeAmount",
>       "msg": "Could not decode amount"
>     },
>     {
>       "code": 6712,
>       "name": "couldNotDecodeRecipient",
>       "msg": "Could not decode recipient"
>     },
>     {
>       "code": 6713,
>       "name": "couldNotDecodeCallbackData",
>       "msg": "Could not decode callback data"
>     },
>     {
>       "code": 6714,
>       "name": "couldNotDecodeContext",
>       "msg": "Could not decode context"
>     },
>     {
>       "code": 6715,
>       "name": "couldNotDecodeFinalAmount",
>       "msg": "Could not decode final amount"
>     },
>     {
>       "code": 6716,
>       "name": "couldNotConvertU256ToU64",
>       "msg": "Could not convert u128 to u64"
>     },
>     {
>       "code": 6717,
>       "name": "couldNotConvertU256ToU128",
>       "msg": "Could not convert u256 to u128"
>     },
>     {
>       "code": 6718,
>       "name": "couldNotConvertU256ToU32",
>       "msg": "Could not convert u256 to u32"
>     },
>     {
>       "code": 6719,
>       "name": "noAttestationsProvided",
>       "msg": "No attestations provided"
>     },
>     {
>       "code": 6720,
>       "name": "notCorrectEncodedTypes",
>       "msg": "Not correct encoded types"
>     },
>     {
>       "code": 6721,
>       "name": "wrongEventSignature",
>       "msg": "Event signature topic does not match OutputFilled or OutputNotFilled"
>     },
>     {
>       "code": 6722,
>       "name": "unauthorizedFillSubmitter",
>       "msg": "Fill submission requires the filler to sign as submitter"
>     }
>   ],
>   "types": [
>     {
>       "name": "chainId",
>       "docs": [
>         "Type used to store the chain id of the chain."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "chainMapping",
>       "docs": [
>         "Type used to map a protocol chain identifier to a custom configured chain identifier."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "oraclePolymer",
>       "docs": [
>         "Represents the `Polymer` oracle implementation type.",
>         "Implements the `OracleBaseTrait`, `InputOracleBaseTrait` and `OutputOracleBaseTrait` traits.",
>         "`polymer_prover_id` is the program id of the polymer prover program.",
>         "`owner` is the owner of the PolymerOracle. The only address to create mappings."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "polymerProverId",
>             "type": "pubkey"
>           },
>           {
>             "name": "owner",
>             "type": "pubkey"
>           },
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "outputProvenEvent",
>       "docs": [
>         "IDL-only mirror of the event emitted by `oracle_base`.",
>         "",
>         "The event is emitted from inside `oracle_base` (see `base.rs`), but",
>         "Anchor's IDL generator only collects `#[event]` structs defined in the",
>         "program crate itself, so without this mirror the event is silently absent",
>         "from the published IDL. Anchor identifies an event by its name-derived",
>         "discriminator (`sha256(\"event:<Name>\")[..8]`) and its borsh field layout,",
>         "so an identical copy here yields the exact IDL entry for the event the",
>         "base crate emits. The test below pins the mirror to the base-crate",
>         "original byte-for-byte."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "remoteIdentifier",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "application",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "payloadHash",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           }
>         ]
>       }
>     }
>   ]
> };
> ```

> [accordion] types/intents\_protocol.ts
>
> ```ts intents_protocol.ts theme={"system"}
> /** Generated by yarn idl:generate. Runtime IDL: ../intents_protocol.json. */
> export type IntentsProtocol = {
>   "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK",
>   "metadata": {
>     "name": "intentsProtocol",
>     "version": "0.0.0",
>     "spec": "0.1.0"
>   },
>   "instructions": [
>     {
>       "name": "consumeLocalAttestation",
>       "docs": [
>         "Consume a local attestation, returning rent immediately unless its",
>         "recorded recipient is executable. In that case preserve a consumed",
>         "account for separate cleanup. Callable only by the consumer committed in the",
>         "attestation's PDA seeds (directly, or via its local-consumer PDA when",
>         "the committed consumer is a program id), and only with the exact",
>         "`data_hash` and `timestamp` the attestation commits to — consuming an",
>         "attestation therefore also validates it."
>       ],
>       "discriminator": [
>         161,
>         130,
>         104,
>         3,
>         101,
>         250,
>         155,
>         239
>       ],
>       "accounts": [
>         {
>           "name": "authority",
>           "docs": [
>             "The committed consumer, or its program's canonical local-consumer PDA."
>           ],
>           "signer": true
>         },
>         {
>           "name": "localAttestation",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   108,
>                   111,
>                   99,
>                   97,
>                   108,
>                   95,
>                   97,
>                   116,
>                   116,
>                   101,
>                   115,
>                   116,
>                   97,
>                   116,
>                   105,
>                   111,
>                   110
>                 ]
>               },
>               {
>                 "kind": "arg",
>                 "path": "attestator"
>               },
>               {
>                 "kind": "arg",
>                 "path": "consumer"
>               },
>               {
>                 "kind": "arg",
>                 "path": "dataHash"
>               }
>             ]
>           }
>         },
>         {
>           "name": "rentRefund",
>           "docs": [
>             "client and CPI interfaces; executable recipients are never modified."
>           ],
>           "writable": true
>         }
>       ],
>       "args": [
>         {
>           "name": "attestator",
>           "type": "pubkey"
>         },
>         {
>           "name": "consumer",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "dataHash",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "timestamp",
>           "type": "u32"
>         }
>       ]
>     },
>     {
>       "name": "createAttestation",
>       "docs": [
>         "Create an attestation for a remote chain."
>       ],
>       "discriminator": [
>         49,
>         24,
>         67,
>         80,
>         12,
>         249,
>         96,
>         239
>       ],
>       "accounts": [
>         {
>           "name": "payer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "attestator",
>           "signer": true
>         },
>         {
>           "name": "attestation",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   97,
>                   116,
>                   116,
>                   101,
>                   115,
>                   116,
>                   97,
>                   116,
>                   105,
>                   111,
>                   110
>                 ]
>               },
>               {
>                 "kind": "account",
>                 "path": "attestator"
>               },
>               {
>                 "kind": "arg",
>                 "path": "chainId"
>               },
>               {
>                 "kind": "arg",
>                 "path": "source"
>               },
>               {
>                 "kind": "arg",
>                 "path": "application"
>               },
>               {
>                 "kind": "arg",
>                 "path": "payloadHash"
>               }
>             ]
>           }
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "chainId",
>           "type": "u128"
>         },
>         {
>           "name": "source",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "application",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "payloadHash",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         }
>       ]
>     },
>     {
>       "name": "createLocalAttestation",
>       "docs": [
>         "Create a local attestation based on the attestation data. The payer is",
>         "recorded as the rent refund destination for later consumption."
>       ],
>       "discriminator": [
>         80,
>         127,
>         2,
>         116,
>         84,
>         21,
>         190,
>         38
>       ],
>       "accounts": [
>         {
>           "name": "payer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "attestator",
>           "signer": true
>         },
>         {
>           "name": "localAttestation",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   108,
>                   111,
>                   99,
>                   97,
>                   108,
>                   95,
>                   97,
>                   116,
>                   116,
>                   101,
>                   115,
>                   116,
>                   97,
>                   116,
>                   105,
>                   111,
>                   110
>                 ]
>               },
>               {
>                 "kind": "account",
>                 "path": "attestator"
>               },
>               {
>                 "kind": "arg",
>                 "path": "consumer"
>               },
>               {
>                 "kind": "arg",
>                 "path": "dataHash"
>               }
>             ]
>           }
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "consumer",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "dataHash",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "timestamp",
>           "type": "u32"
>         }
>       ]
>     },
>     {
>       "name": "reclaimLocalAttestationRent",
>       "docs": [
>         "Reclaim deferred rent from a consumed attestation. Anyone may call this;",
>         "rent goes to the recorded payer unless that payer signs to authorize",
>         "another destination. The destination must be non-executable."
>       ],
>       "discriminator": [
>         124,
>         240,
>         79,
>         198,
>         173,
>         188,
>         63,
>         76
>       ],
>       "accounts": [
>         {
>           "name": "localAttestation",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   108,
>                   111,
>                   99,
>                   97,
>                   108,
>                   95,
>                   97,
>                   116,
>                   116,
>                   101,
>                   115,
>                   116,
>                   97,
>                   116,
>                   105,
>                   111,
>                   110
>                 ]
>               },
>               {
>                 "kind": "arg",
>                 "path": "attestator"
>               },
>               {
>                 "kind": "arg",
>                 "path": "consumer"
>               },
>               {
>                 "kind": "arg",
>                 "path": "dataHash"
>               }
>             ]
>           }
>         },
>         {
>           "name": "rentRefund",
>           "writable": true
>         },
>         {
>           "name": "rentRefundAuthority",
>           "docs": [
>             "Required only when returning rent to an address other than the payer.",
>             "A payer converted into a program can still sign using its original key."
>           ],
>           "signer": true,
>           "optional": true
>         }
>       ],
>       "args": [
>         {
>           "name": "attestator",
>           "type": "pubkey"
>         },
>         {
>           "name": "consumer",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         },
>         {
>           "name": "dataHash",
>           "type": {
>             "array": [
>               "u8",
>               32
>             ]
>           }
>         }
>       ]
>     },
>     {
>       "name": "setChainId",
>       "docs": [
>         "Set the chain id of the chain."
>       ],
>       "discriminator": [
>         57,
>         174,
>         29,
>         202,
>         84,
>         216,
>         7,
>         148
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "chainId",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   99,
>                   104,
>                   97,
>                   105,
>                   110,
>                   95,
>                   105,
>                   100
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "chainId",
>           "type": "u128"
>         }
>       ]
>     },
>     {
>       "name": "setChainMapping",
>       "docs": [
>         "Set the real chain id for a protocol chain id.",
>         "Dev:",
>         "`protocol_chain_id` is the protocol chain identifier.",
>         "`chain_id` is the real chain identifier."
>       ],
>       "discriminator": [
>         174,
>         145,
>         44,
>         171,
>         203,
>         101,
>         230,
>         130
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oracle",
>           "signer": true
>         },
>         {
>           "name": "chainMapping",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   99,
>                   104,
>                   97,
>                   105,
>                   110,
>                   95,
>                   109,
>                   97,
>                   112,
>                   112,
>                   105,
>                   110,
>                   103
>                 ]
>               },
>               {
>                 "kind": "account",
>                 "path": "oracle"
>               },
>               {
>                 "kind": "arg",
>                 "path": "protocolChainId"
>               }
>             ]
>           }
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "protocolChainId",
>           "type": "u128"
>         },
>         {
>           "name": "chainId",
>           "type": "u128"
>         }
>       ]
>     }
>   ],
>   "accounts": [
>     {
>       "name": "attestation",
>       "discriminator": [
>         152,
>         125,
>         183,
>         86,
>         36,
>         146,
>         121,
>         73
>       ]
>     },
>     {
>       "name": "chainId",
>       "discriminator": [
>         203,
>         146,
>         201,
>         161,
>         230,
>         71,
>         157,
>         218
>       ]
>     },
>     {
>       "name": "chainMapping",
>       "discriminator": [
>         81,
>         78,
>         77,
>         209,
>         44,
>         206,
>         85,
>         169
>       ]
>     },
>     {
>       "name": "localAttestation",
>       "discriminator": [
>         173,
>         247,
>         101,
>         168,
>         96,
>         238,
>         17,
>         22
>       ]
>     }
>   ],
>   "events": [
>     {
>       "name": "attestationCreatedEvent",
>       "discriminator": [
>         157,
>         10,
>         208,
>         90,
>         87,
>         62,
>         175,
>         26
>       ]
>     },
>     {
>       "name": "chainIdConfiguredEvent",
>       "discriminator": [
>         92,
>         39,
>         243,
>         11,
>         30,
>         97,
>         80,
>         219
>       ]
>     },
>     {
>       "name": "chainMapConfiguredEvent",
>       "discriminator": [
>         157,
>         35,
>         137,
>         206,
>         200,
>         100,
>         92,
>         171
>       ]
>     },
>     {
>       "name": "localAttestationConsumedEvent",
>       "discriminator": [
>         238,
>         77,
>         179,
>         71,
>         131,
>         58,
>         164,
>         104
>       ]
>     },
>     {
>       "name": "localAttestationCreatedEvent",
>       "discriminator": [
>         173,
>         59,
>         45,
>         101,
>         206,
>         241,
>         125,
>         25
>       ]
>     },
>     {
>       "name": "localAttestationRentDeferredEvent",
>       "discriminator": [
>         20,
>         52,
>         250,
>         84,
>         165,
>         187,
>         163,
>         219
>       ]
>     },
>     {
>       "name": "localAttestationRentReclaimedEvent",
>       "discriminator": [
>         142,
>         21,
>         161,
>         230,
>         143,
>         239,
>         210,
>         124
>       ]
>     }
>   ],
>   "errors": [
>     {
>       "code": 6100,
>       "name": "invalidOracle",
>       "msg": "Invalid oracle"
>     },
>     {
>       "code": 6101,
>       "name": "invalidLocalAttestationTimestamp",
>       "msg": "Invalid local attestation timestamp"
>     },
>     {
>       "code": 6102,
>       "name": "invalidChainMappingAccount",
>       "msg": "Invalid chain mapping account"
>     },
>     {
>       "code": 6103,
>       "name": "invalidLocalAttestationAccount",
>       "msg": "Invalid local attestation account"
>     },
>     {
>       "code": 6104,
>       "name": "invalidAttestationAccount",
>       "msg": "Invalid attestation account"
>     },
>     {
>       "code": 6105,
>       "name": "invalidChainIdAccount",
>       "msg": "Invalid chain id account"
>     },
>     {
>       "code": 6106,
>       "name": "unauthorizedLocalAttestationConsumer",
>       "msg": "Signer is not the attestation's committed consumer"
>     },
>     {
>       "code": 6107,
>       "name": "invalidRentRefundAccount",
>       "msg": "Rent refund account does not match the stored rent refund"
>     },
>     {
>       "code": 6108,
>       "name": "localAttestationAlreadyConsumed",
>       "msg": "Local attestation has already been consumed"
>     },
>     {
>       "code": 6109,
>       "name": "executableRentRefundAccount",
>       "msg": "Rent refund recipient is still executable"
>     },
>     {
>       "code": 6110,
>       "name": "localAttestationNotConsumed",
>       "msg": "Local attestation has not been consumed"
>     }
>   ],
>   "types": [
>     {
>       "name": "attestation",
>       "docs": [
>         "Type used to indicate that some data from another chain data has been attested.",
>         "It is derived from the attestation data as seeds"
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "attestationCreatedEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "attestator",
>             "type": "pubkey"
>           },
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "source",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "application",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "payloadHash",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "chainId",
>       "docs": [
>         "Type used to store the chain id of the chain."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "chainIdConfiguredEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainIdAccountAddress",
>             "type": "pubkey"
>           },
>           {
>             "name": "chainId",
>             "type": "u128"
>           }
>         ]
>       }
>     },
>     {
>       "name": "chainMapConfiguredEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "oracle",
>             "type": "pubkey"
>           },
>           {
>             "name": "protocolChainId",
>             "type": "u128"
>           },
>           {
>             "name": "chainId",
>             "type": "u128"
>           }
>         ]
>       }
>     },
>     {
>       "name": "chainMapping",
>       "docs": [
>         "Type used to map a protocol chain identifier to a custom configured chain identifier."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "localAttestation",
>       "docs": [
>         "Type used to indicate that some data has been attested by an attestator.",
>         "It is derived from the local attestation data as seeds",
>         "`timestamp` is the timestamp of the local attestation was created at.",
>         "It is stored within the account since it cannot be known before creation.",
>         "`rent_refund` is the payer that funded the account; it receives the lamports",
>         "when consumed, or during deferred cleanup if that payer is executable."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "timestamp",
>             "type": "u32"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           },
>           {
>             "name": "rentRefund",
>             "type": "pubkey"
>           },
>           {
>             "name": "consumed",
>             "docs": [
>               "Consumed proofs retain their rent until cleanup when the payer is executable."
>             ],
>             "type": "bool"
>           }
>         ]
>       }
>     },
>     {
>       "name": "localAttestationConsumedEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "attestator",
>             "type": "pubkey"
>           },
>           {
>             "name": "consumer",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "dataHash",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "rentRefund",
>             "type": "pubkey"
>           }
>         ]
>       }
>     },
>     {
>       "name": "localAttestationCreatedEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "attestator",
>             "type": "pubkey"
>           },
>           {
>             "name": "consumer",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "timestamp",
>             "type": "u32"
>           },
>           {
>             "name": "dataHash",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           }
>         ]
>       }
>     },
>     {
>       "name": "localAttestationRentDeferredEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "localAttestation",
>             "type": "pubkey"
>           },
>           {
>             "name": "rentRefund",
>             "type": "pubkey"
>           }
>         ]
>       }
>     },
>     {
>       "name": "localAttestationRentReclaimedEvent",
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "localAttestation",
>             "type": "pubkey"
>           },
>           {
>             "name": "rentRefund",
>             "type": "pubkey"
>           }
>         ]
>       }
>     }
>   ]
> };
> ```

> [accordion] types/wormhole.ts
>
> ```ts wormhole.ts theme={"system"}
> /** Generated by yarn idl:generate. Runtime IDL: ../wormhole.json. */
> export type Wormhole = {
>   "address": "Fy28CqdB1WtDJLdc18fDGKqvpG8HxtSiSG5HChdhR4nU",
>   "metadata": {
>     "name": "wormhole",
>     "version": "0.0.0",
>     "spec": "0.1.0"
>   },
>   "instructions": [
>     {
>       "name": "initialize",
>       "docs": [
>         "Initializes the `oracle_wormhole` account. Creates `oracle_wormhole` PDA and sets the `core_bridge_id`.",
>         "`oracle_wormhole` PDA is used to store the chain identifier, the owner address, and the Wormhole core bridge program id."
>       ],
>       "discriminator": [
>         175,
>         175,
>         109,
>         31,
>         13,
>         152,
>         155,
>         237
>       ],
>       "accounts": [
>         {
>           "name": "deployer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "chainId"
>         },
>         {
>           "name": "oracleWormhole",
>           "writable": true,
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   119,
>                   111,
>                   114,
>                   109,
>                   104,
>                   111,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "coreBridgeId",
>           "type": "pubkey"
>         }
>       ]
>     },
>     {
>       "name": "receive",
>       "docs": [
>         "Receive attestation proofs from a remote oracle and register them",
>         "locally by creating `attestation` accounts.",
>         "Remaining accounts: exactly one writable destination attestation PDA per decoded VAA payload, in decoded payload order. No remaining-account signers or refund slots; the named signer funds account creation.",
>         "Derive each address with the intents protocol program ID and seeds, in order: UTF-8 'attestation'; this program's oracle_wormhole state PDA (the attestator); the canonical chain ID as u128 little-endian, obtained by mapping the VAA emitter chain through the chain_mapping account (not the raw Wormhole chain ID); the 32-byte VAA emitter address (source); the 32-byte application from the decoded message; keccak256 of that payload."
>       ],
>       "discriminator": [
>         86,
>         17,
>         255,
>         171,
>         17,
>         17,
>         187,
>         219
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oracleWormhole",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   119,
>                   111,
>                   114,
>                   109,
>                   104,
>                   111,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "coreBridgeProgram"
>         },
>         {
>           "name": "postedVaa"
>         },
>         {
>           "name": "chainMapping"
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": []
>     },
>     {
>       "name": "setChainMapping",
>       "docs": [
>         "Maps wormhole chain id to a configured custom chain id.",
>         "Can only be called by the owner of the `oracle_wormhole` account.",
>         "Each mapping may only be set once."
>       ],
>       "discriminator": [
>         174,
>         145,
>         44,
>         171,
>         203,
>         101,
>         230,
>         130
>       ],
>       "accounts": [
>         {
>           "name": "signer",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oracleWormhole",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   119,
>                   111,
>                   114,
>                   109,
>                   104,
>                   111,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "chainMapping",
>           "writable": true
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "protocolChainId",
>           "type": "u128"
>         },
>         {
>           "name": "chainId",
>           "type": "u128"
>         }
>       ]
>     },
>     {
>       "name": "submit",
>       "docs": [
>         "Validates the proofs and posts a message via Wormhole.",
>         "Remaining accounts: exactly [local_attestation_0, rent_refund_0, ..., local_attestation_N-1, rent_refund_N-1] in fill_descriptions order. Payloads must not be empty. Both accounts in every pair must be writable; no additional remaining-account signers.",
>         "Each rent_refund must be the attestation's stored recipient. Executable recipients defer reclamation; keep both writable. The named submitter and fresh post_message account must sign; there is no additional filler-signature requirement."
>       ],
>       "discriminator": [
>         88,
>         166,
>         102,
>         181,
>         162,
>         127,
>         170,
>         48
>       ],
>       "accounts": [
>         {
>           "name": "submitter",
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "oracleWormhole",
>           "pda": {
>             "seeds": [
>               {
>                 "kind": "const",
>                 "value": [
>                   119,
>                   111,
>                   114,
>                   109,
>                   104,
>                   111,
>                   108,
>                   101
>                 ]
>               }
>             ]
>           }
>         },
>         {
>           "name": "postMessage",
>           "docs": [
>             "Fresh client-generated account that the Wormhole core bridge creates and",
>             "owns during the CPI. The core bridge requires the message account to sign,",
>             "so it is declared as a `Signer`. Using a unique account per submit (instead",
>             "of a shared, sequence-derived PDA) guarantees that independent or concurrent",
>             "submits can never collide on the same address."
>           ],
>           "writable": true,
>           "signer": true
>         },
>         {
>           "name": "intentsProtocolProgram",
>           "address": "LiFixdGLT5CMdLsHBvaijTXpPy4Uux9Y53SXkuR4HaK"
>         },
>         {
>           "name": "coreBridgeProgram"
>         },
>         {
>           "name": "wormholeConfig",
>           "writable": true
>         },
>         {
>           "name": "wormholeSequence",
>           "writable": true
>         },
>         {
>           "name": "wormholeFeeCollector",
>           "writable": true
>         },
>         {
>           "name": "clock",
>           "address": "SysvarC1ock11111111111111111111111111111111"
>         },
>         {
>           "name": "rent",
>           "address": "SysvarRent111111111111111111111111111111111"
>         },
>         {
>           "name": "systemProgram",
>           "address": "11111111111111111111111111111111"
>         }
>       ],
>       "args": [
>         {
>           "name": "source",
>           "type": "pubkey"
>         },
>         {
>           "name": "fillDescriptions",
>           "type": {
>             "vec": "bytes"
>           }
>         }
>       ]
>     }
>   ],
>   "accounts": [
>     {
>       "name": "chainId",
>       "discriminator": [
>         203,
>         146,
>         201,
>         161,
>         230,
>         71,
>         157,
>         218
>       ]
>     },
>     {
>       "name": "chainMapping",
>       "discriminator": [
>         81,
>         78,
>         77,
>         209,
>         44,
>         206,
>         85,
>         169
>       ]
>     },
>     {
>       "name": "oracleWormhole",
>       "discriminator": [
>         218,
>         207,
>         238,
>         190,
>         98,
>         163,
>         3,
>         199
>       ]
>     }
>   ],
>   "errors": [
>     {
>       "code": 6800,
>       "name": "wrongProgram",
>       "msg": "Wrong program"
>     },
>     {
>       "code": 6801,
>       "name": "invalidPostedVaa",
>       "msg": "Invalid posted VAA data"
>     },
>     {
>       "code": 6802,
>       "name": "invalidChainId",
>       "msg": "Invalid chain id"
>     },
>     {
>       "code": 6803,
>       "name": "unauthorized",
>       "msg": "Unauthorized mapping creation"
>     }
>   ],
>   "types": [
>     {
>       "name": "chainId",
>       "docs": [
>         "Type used to store the chain id of the chain."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "chainMapping",
>       "docs": [
>         "Type used to map a protocol chain identifier to a custom configured chain identifier."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "oracleWormhole",
>       "docs": [
>         "Represents the `Wormhole` oracle implementation type.",
>         "Implements the `OracleBaseTrait`, `InputOracleBaseTrait` and `OutputOracleBaseTrait` traits.",
>         "`core_bridge_id` is the program id of the Wormhole core bridge program.",
>         "`owner` is the owner of the WormholeOracle. The only address to create mappings."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "owner",
>             "type": "pubkey"
>           },
>           {
>             "name": "coreBridgeId",
>             "type": "pubkey"
>           },
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "bump",
>             "type": "u8"
>           }
>         ]
>       }
>     },
>     {
>       "name": "outputProvenEvent",
>       "docs": [
>         "IDL-only mirror of the event emitted by `oracle_base`.",
>         "",
>         "The event is emitted from inside `oracle_base` (see `base.rs`), but",
>         "Anchor's IDL generator only collects `#[event]` structs defined in the",
>         "program crate itself, so without this mirror the event is silently absent",
>         "from the published IDL. Anchor identifies an event by its name-derived",
>         "discriminator (`sha256(\"event:<Name>\")[..8]`) and its borsh field layout,",
>         "so an identical copy here yields the exact IDL entry for the event the",
>         "base crate emits. The test below pins the mirror to the base-crate",
>         "original byte-for-byte."
>       ],
>       "type": {
>         "kind": "struct",
>         "fields": [
>           {
>             "name": "chainId",
>             "type": "u128"
>           },
>           {
>             "name": "remoteIdentifier",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "application",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           },
>           {
>             "name": "payloadHash",
>             "type": {
>               "array": [
>                 "u8",
>                 32
>               ]
>             }
>           }
>         ]
>       }
>     }
>   ],
>   "events": [
>     {
>       "name": "outputProvenEvent",
>       "discriminator": [
>         85,
>         190,
>         185,
>         8,
>         90,
>         48,
>         3,
>         214
>       ]
>     }
>   ]
> };
> ```

## Next steps

> [card] [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md)
>
> What each program does, and its account model

> [card] [Debugging on Solana](https://docs.li.fi/lifi-intents/knowledge-database/solana-debugging.md)
>
> Read failed transactions and error codes


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.