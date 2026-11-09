> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Solana vs EVM

> What an integrator or solver must do differently to use LI.FI Intents on Solana compared to an EVM chain: platform concepts, addresses, endpoints, settlement, and proving.

Solana is a supported Intents chain (mainnet `solana:1151111081099710`). It is not an EVM-compatible runtime: programs replace contracts, accounts replace storage, and rent, PDAs, and transaction size limits shape how orders work. The high-level mental model is unchanged — input settlement, output settlement, oracle validation — but several mechanics differ.

This page is a delta reference: for each concern, what EVM does and what Solana does instead. It starts with a short platform primer for readers coming from EVM.

> [!NOTE]
>
> For general (non-Intents) Solana support across LI.FI products, see the [Solana ecosystem page](https://docs.li.fi/introduction/solana-ecosystem.md).

***

## At a glance

The table summarizes every delta on this page; the sections below explain each one.

| Concern | EVM | Solana |
| - | - | - |
| Address format | `0x` hex, 20 bytes | base58, 32 bytes |
| Settlement types | Escrow **and** Compact | **Escrow only** |
| Order open path | On-chain open **or** gasless `POST /orders/submit` | **On-chain open only** (sponsored, never gasless) |
| Inputs per order | Multiple (`uint256[2][]`) | **One** (`MandateInput`) |
| Amount width | `uint256` | **`u64`** (SPL limit) |
| Order size | Practically unbounded | **`finalise` must fit one transaction** (enforced at `open`) |
| Proof storage | Contract storage | **PDA accounts** |
| Batch fills | `fillOrderOutputs` fills a whole order | One instruction **per output** |
| Input-first same-chain settlement | Callback from `finalise` | **Pinned later fill**, single output only |
| Calls on delivery | Supported | **Not supported** |

***

## Solana in sixty seconds

The Solana pages lean on a few platform fundamentals. If you come from an EVM background, this is enough to follow along; each item links to the official docs for depth.

* **Program Derived Address (PDA).** An account address derived deterministically from a program id and a list of seeds. PDAs have no private key — only the deriving program can sign for them. The intent system uses PDAs everywhere: escrows, attestations, and configuration all live at addresses anyone can recompute, which is why "the account exists at the expected PDA" can serve as a proof. [Solana docs: PDAs](https://solana.com/docs/core/pda)
* **Cross-program invocation (CPI).** One program calling an instruction on another within the same transaction — Solana's equivalent of a contract calling a contract. The output settler creates attestation accounts by CPI into the `intents_protocol` program. [Solana docs: CPI](https://solana.com/docs/core/cpi)
* **Associated token account (ATA) and rent.** SPL token balances live in separate token accounts, not in the wallet address itself; the ATA is the canonical token account for an (owner, mint) pair. Every account must hold a small SOL deposit ("rent") to exist — whoever creates an account pays the rent and gets it back when the account closes. This is why orders track a *sponsor* (who paid rent at `open` and is repaid the escrow token account and `order_context` rent when the order closes — the `consumed_order` rent is a permanent per-order cost), why solvers may pay rent when a recipient's ATA does not exist yet, and why fill records and attestations record who paid for them so that rent can be returned later. [SPL docs: ATAs](https://spl.solana.com/associated-token-account)
* **Anchor.** The framework the intent programs are written in. It declares each instruction's accounts, generates an IDL (comparable to an EVM ABI) for clients, and prefixes events and accounts with discriminators. The TypeScript examples use Anchor's client library, `@coral-xyz/anchor`. [Anchor docs](https://www.anchor-lang.com/docs)
* **`remaining_accounts`.** Anchor instructions declare a fixed account list, but callers can append extra accounts at the end. The intent system uses them for lists whose length depends on the order — such as one attestation account plus one rent-refund account per output at `finalise`.
* **Borsh.** The binary serialization format Anchor uses for instruction data, accounts, and events. Order ids derive from it: `order_id = keccak256(borsh_serialize(StandardOrder))`. [Borsh](https://borsh.io/)
* **Base58 and bytes32.** Solana addresses are 32-byte ed25519 public keys displayed as base58 strings (e.g. `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`). The cross-chain order format stores all addresses as raw 32-byte values: a Solana address is the base58-decoded 32 bytes, while a 20-byte EVM address is left-padded with zeros.
* **Verifiable action approval (VAA).** Wormhole's signed message format — an attestation by Wormhole's guardian network that an event happened on a source chain. The Wormhole oracle posts fill proofs as VAAs. [Wormhole docs: VAAs](https://wormhole.com/docs/protocol/infrastructure/vaas/)

***

## Chain IDs

In API requests, Solana chains use CAIP-2-derived ids in the `solana` namespace:

| Cluster | Chain id |
| - | - |
| Solana Mainnet | `solana:1151111081099710` |
| Solana Devnet | `solana:1151111081099712` |

On-chain, the protocol uses a `u128` chain id stored in the `intents_protocol` `ChainId` PDA. That value — not the EVM-style native `chainId` — is what appears in `StandardOrder.origin_chain_id` and `MandateOutput.chain_id`, and what programs validate against.

Solana has no single "chain id" — three namespaces coexist, and each surface expects exactly one of them:

| Namespace | Solana mainnet value | Where it appears |
| - | - | - |
| CAIP-2 / API id | `solana:1151111081099710` (bare `1151111081099710` on the solver quote endpoint) | API requests and responses |
| Canonical on-chain id | `1151111081099710` as a `u128` | `StandardOrder.origin_chain_id`, `MandateOutput.chain_id`, the `ChainId` PDA |
| Polymer protocol id | `2` | Polymer's Prove API (`srcChainId`) and the oracle's chain mappings |

Using a value from the wrong namespace fails indirectly. Quoting a Solana leg under `eip155:` is rejected with an error about the *asset encoding* rather than the namespace. A Polymer proof requested with the canonical id instead of `2` is refused by the Prove API — see [Oracle systems](https://docs.li.fi/lifi-intents/architecture/oracle-systems.md#polymer-on-solana) for the Polymer flow.

## Quoting

Solana is served by the standard quote endpoint, `POST /api/v1/integrator/quote/request` — the same one EVM and Tron use. Pass Solana addresses as base58 with the `solana:` CAIP-2 chain id and get base58 back. Request and response examples are in [Request a Quote](https://docs.li.fi/lifi-intents/intents-api/request-quote.md#solana-request). Solana is not served by the legacy V0 endpoint.

For a Solana-origin order, the winning quote returns a ready-to-open transaction rather than an order to submit. The `order.openIntentTx` fields differ from EVM:

* `to` is the input settler program id, in base58.
* `data` is a **base58-encoded, serialized v0 `VersionedTransaction`**, not calldata. Its `recentBlockhash` is a dummy all-zeros value (`11111111111111111111111111111111`) and must be replaced with a fresh blockhash before signing — Solana blockhashes expire after roughly a minute.
* `computeUnitsRequired` estimates the compute units the transaction needs (a decimal string). The serialized template does not contain a Compute Budget instruction; rebuild the message with `ComputeBudgetProgram.setComputeUnitLimit` before signing, as shown in [Opening on Solana](https://docs.li.fi/lifi-intents/intents-api/create-and-submit.md#opening-on-solana).
* `checks.allowances` is always `[]`: the SPL transfer happens inside the `open` instruction, so there is no approval pre-step.
* `metadata.exclusiveFor`, when set, is the input-chain solver identity with the exclusive right to fill during the exclusivity window. For a Solana-origin order it must be the base58 Solana public key that will sign `finalise`, even when the output is on EVM.

## Token support

| Token kind | As input | As output | Notes |
| - | - | - | - |
| Classic SPL | ✓ | ✓ | Supported by the quote endpoint |
| Native SOL | ✗ | ✓ | `open` requires an SPL mint as input. As an output, encode the token as 32 zero bytes; solvers deliver via `native_fill` |
| Wrapped SOL (WSOL) | ✓ | ✓ | Treated as a normal SPL mint — nothing wraps or unwraps automatically on either side |
| Compatible Token-2022 | ✓ | ✓ | Supported unless the mint has an active transfer hook, a non-zero transfer fee, or the non-transferable extension |

xStocks use Token-2022 and can use the standard quote flow when a specific mint's current configuration is compatible. Compatibility is evaluated per mint against its on-chain configuration. A mint is rejected when any of the following holds:

* **Active transfer hook** — the transfer-hook extension has a hook program set. The extension may be present with no program configured, in which case the mint remains compatible.
* **Non-zero transfer fee** — the transfer-fee extension is configured with a fee above zero in either the current or the scheduled epoch. A zero-fee configuration (as on PYUSD) is compatible.
* **Non-transferable** — the mint carries the non-transferable extension, so no transfer can ever succeed.
* **Unresolvable mint** — the mint account does not exist or is not owned by a known token program.

Other extensions, including ones the service does not yet recognize, do not by themselves cause a mint to be rejected. This eligibility check is not a guarantee that every transfer will succeed: issuer-controlled restrictions such as pausing or freezing can still make simulation or execution fail. Transfer-hook programs and fee rates can also change, so a mint's compatibility can change over time.

The same policy is enforced twice: the quote endpoint returns an empty quote list for an incompatible mint (or when the mint cannot be read), and orders opened on-chain that reference an incompatible mint are not distributed to solvers.

## Settlement: escrow only

Solana supports only the escrow input settler (`input_settler_escrow`). There is no Compact or resource-lock flow, so the [Compact Orders](https://docs.li.fi/lifi-intents/intents-api/compact-orders.md) guide does not apply to Solana.

## One input per order

A Solana order carries exactly one input — a single `{ token, amount }` pair (`MandateInput`) — where an EVM order can carry an array of inputs. If you need to spend several tokens, you need several orders.

## Amounts are u64

Solana narrows the wire-format 32-byte amount and Dutch-auction `slope` fields to `u64`, the SPL token limit; EVM treats them as full-width `uint256`. When you build cross-chain Dutch orders destined for Solana, keep the slope and the maximum resolved amount within `u64` range, or the output cannot be filled. See [Auctions](https://docs.li.fi/lifi-intents/for-solvers/auctions.md) for the pricing formula.

## Order size

On EVM, order size is practically unbounded. On Solana, the whole `StandardOrder` travels in the instruction data of `open` and `finalise`, and `finalise` adds an attestation account and a rent-refund account per output — all inside Solana's 1,232-byte transaction packet.

The escrow enforces the bound at `open`: it projects the finalise transaction as `760 + 302·N + Σ(|callback_data| + |context|)` bytes for `N` outputs and rejects the order (`FinaliseTxTooLarge`) above 1,142 bytes — the packet cap minus headroom reserved for compute-budget instructions, with 32 bytes credited back when the sponsor is the user. An order that opens can therefore always be finalised with a bare finalise transaction; re-check the bound only if you append instructions of your own.

The practical consequence: **an order opened on Solana has exactly one output.** One output projects to 1,062 bytes (1,030 when the sponsor is the user), leaving 80 bytes (112) for `callback_data` and `context` — enough for every order type when there is no callback data (the largest context, exclusive Dutch auction, is 73 bytes). Two outputs project to at least 1,332 bytes and are always rejected. Orders that originate on another chain are not limited this way and can deliver several outputs on Solana.

## Proofs are accounts

EVM stores fill proofs in contract storage behind validation interfaces. Solana stores them as deterministic PDA accounts: a `LocalAttestation` for a fill on Solana, an `Attestation` for a remote proof registered by an oracle. The full model — seeds, payload encodings, and the end-to-end flow — is in [Attestations](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#attestations).

## Same-chain swaps

Both platforms support same-chain intents, implemented differently.

On Solana, these orders put the output-settler state PDA in all three identity slots (`settler`, `oracle`, `inputOracle`) — see the [identity matrix](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#which-identity-goes-in-which-field). On the standard route, `fill` creates a `LocalAttestation` and finalization validates that account directly — no oracle program involved. The escrow is not the attestation's consumer under this encoding, so the attestation's rent is not returned. On the input-first route below, no attestation is created at all.

On EVM, same-chain is modeled as an oracle configuration (the output settler doubles as the oracle).

Both platforms can release the input before output delivery. Their execution models differ:

| | EVM | Solana |
| - | - | - |
| Mechanism | `orderFinalised` callback from `finalise`, run after inputs are paid and before proofs are checked | `finalise_with_prefill`, which requires a matching compact fill later in the same transaction |
| Outputs | One or more same-chain outputs | Exactly one same-chain output |
| Where the strategy lives | The solver's own contract, called back | Plain instructions between the two settlement instructions; neither may be called by a program |
| Pricing types | All | Limit and exclusive limit only |
| Hard limit | Block gas limit | One 1,232-byte transaction |

See [Input-first Solana settlement](https://docs.li.fi/lifi-intents/for-solvers/settlement.md#input-first-solana-settlement) for the solver workflow and constraints.

## One fill per instruction

There is no Solana equivalent of `fillOrderOutputs` that atomically fills several outputs at once. You fill each output with its own instruction, one after another. This matters for orders from another chain with several Solana outputs; an order opened on Solana has a single output.

***

## Next steps

> [card] [Solana programs](https://docs.li.fi/lifi-intents/architecture/solana-programs.md)
>
> The Anchor programs, structs, and instructions behind Solana intents


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.