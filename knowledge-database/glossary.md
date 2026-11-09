> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Glossary

> Intent systems and cross-chain bridges use many specialized terms. This page explains arbiters, allocators, sponsors, locks, attestations, settlers, and more.

## Allocator

An entity that validates that a lock does not exceed a user's current balance. It may also act as an arbiter. The user chooses an allocator when depositing into a lock.

## Arbiter

An entity, contract, or mechanism that can determine when a resource lock's desired action has been fulfilled. It may also act as an allocator.

## ATA

Associated token account — the canonical SPL token account for an (owner, mint) pair on Solana. See the [Solana primer](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md#solana-in-sixty-seconds).

## Attestation

On Solana, a deterministic PDA account created when an input-side oracle verifies a remote proof or message. Its existence at the expected address is the proof used for cross-chain finalization. See [Attestations](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#attestations).

## Canonical solver

The solver of the first output in an order. The canonical solver receives the full escrowed input regardless of who filled the other outputs. On Solana, only the canonical solver may call `finalise`.

## CPI

Cross-program invocation — one Solana program calling an instruction on another within the same transaction; Solana's equivalent of a contract calling a contract. See the [Solana primer](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md#solana-in-sixty-seconds).

## Emissary

For smart wallets unable to provide stable signatures, an emissary can be used to produce stable signatures. The emissary is usually a trusted entity, meaning that once a signature is issued it cannot be redacted. The terminology is specific to The Compact.

## Fill first

A swap flow where the tokens are delivered to the user *before* the input tokens are collected by a protocol. These flows use specialized wallet techniques like [resource locks](#resource-lock) to remain safe.

## FillRecord

On Solana, the PDA the output settler creates on every successful fill (formerly `FillId`). Its existence blocks a second fill of the same output. It records the filler who paid its rent and a `close_after` time 48 hours after the fill deadline; after that, anyone can close it to return the rent to the filler. See [Fill records](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#fill-records).

## GMP

Generalized Message Passing. Describes a method to send messages between two chains. Examples include Wormhole, LayerZero, Hyperlane, and more.

## Input-first settlement

On Solana, a single-transaction route that releases a same-chain order's input before its required output fill. See [Input-first Solana settlement](https://docs.li.fi/lifi-intents/for-solvers/settlement.md#input-first-solana-settlement).

## Input

The starting point. Input assets refer to the assets paid into the intent system. The input chain refers to the chain(s) of the input assets.

## Input settler

A contract or program on the origin chain that verifies proofs of delivery and releases locked funds to the solver. In a resource-lock-based system, the input settler often acts as an arbiter.

## Integrator

See [Intent issuer](#intent-issuer).

## Intent

A statement of the action a user wants. Intents are often used to describe swaps but can also describe desired cross-chain interactions. Unlike swaps, intents should generally be composable. Intents are usually self-contained, meaning they describe both the desired end state and the payment to achieve that state; the output and input, respectively.

## Intent issuer

Someone who issues intents for a cross-chain intent system. The intent issuer specifies how the intent is configured. It is then up to [solvers](#solver) to determine whether or not they will fill the intent.

## LocalAttestation

On Solana, a deterministic PDA account created when an output is filled with `fill` or `native_fill` (or attested as not filled) on Solana itself; the compact same-chain fills do not create one. It stores the fill timestamp, the account that paid its rent, and a `consumed` flag. Used for same-chain finalization and output-side oracle validation. Its committed consumer can consume it once, which invalidates the proof and returns the rent (or defers it, if the payer is an executable account). See [Attestations](https://docs.li.fi/lifi-intents/architecture/solana-programs.md#attestations).

## Lock

An allowance issued by a user to an arbiter. Before locks are valid, they must be signed by the user and co-signed by the allocator.

## Non-fill attestation

An attestation proving an output was **not** filled before its fill deadline. Anyone can create it once the deadline passes — on Solana, only within the 48 hours after the deadline; the input chain uses it to refund the order early, without waiting for expiry.

## Optimistic

A validation method that assumes statements are true unless disputed.

## Oracle

A contract or program that verifies whether solver delivery conditions were met so settlement can proceed. For same-chain intents, the output settler can be used directly as the oracle. Oracles may use optimistic systems, messaging bridges, light clients, and other mechanisms.

## Order ID

The canonical identifier of an order. On Solana it is derived from the order's content: `order_id = keccak256(borsh_serialize(StandardOrder))`.

## Order server

A server that sits between intent issuers and solvers, and helps broadcast and distribute intents. LI.FI's order server (`https://order.li.fi`) collects solver quotes, builds open transactions for integrators, ingests on-chain events, and broadcasts orders to solvers.

## Output

The endpoint. Output assets refer to the desired tokens to be paid to the user (or used within a larger action). The output chain refers to the chain(s) of the output assets.

## Output settler

A contract or program that accepts solver fills on destination chains and generates the attestations the oracle and the input settler use to verify a fill.

## PDA

Program Derived Address — a deterministic, program-owned Solana account address derived from seeds. The primitive behind the intent system's escrows, attestations, and configuration on Solana. See the [Solana primer](https://docs.li.fi/lifi-intents/knowledge-database/solana-deltas.md#solana-in-sixty-seconds).

## Resource lock

An umbrella concept for lock-based input settlement mechanisms (for example escrow and Compact) that hold user funds until settlement conditions are met. You can find a primer in our [knowledge hub](https://docs.li.fi/lifi-intents/knowledge-database/resource-locks.md).

## Solver

A specialized third party who fulfills cross-chain intents using a variety of liquidity sources. They earn the margin between what reaching the end state costs them and what the inputs are worth.

## Sponsor

The owner of the tokens to be used for locks in a resource lock. On Solana the word means something different: the sponsor is the account that signs `open` to pay the rent for the order's accounts — the user themselves or a third party. It need not be the transaction fee payer. The escrow token account and `order_context` rent are returned to it when the order closes (finalise or refund), or to the settlement signer if the sponsor is an executable account; the `consumed_order` rent is a permanent per-order cost.

## Validation layer

An entity that can validate whether an event actually happened. Examples include GMP, optimistic proof, or bridges.

## Exclusive Window

The time after an order opens when only its exclusive solver can fill it. It ends at the order's exclusive deadline, or at its fill deadline if that comes first. See [Auctions](https://docs.li.fi/lifi-intents/for-solvers/auctions.md).

## Missed Deadline

An order a solver won that was not filled on time and counts against it, for example because nobody filled it. Missed deadlines can pause a solver. See [Reputation](https://docs.li.fi/lifi-intents/for-solvers/reputation.md#orders-that-count) for which results count as a miss.

## Pause

A period when the order server does not select a solver's quotes, for one destination chain or for every chain. Solvers are paused automatically after too many [missed deadlines](#missed-deadline). See [Reputation](https://docs.li.fi/lifi-intents/for-solvers/reputation.md#pauses).

## Recovery Trial

The step after a pause, where a solver wins a limited number of orders until it has filled enough of them on time. See [Reputation](https://docs.li.fi/lifi-intents/for-solvers/reputation.md#recovery-trials).


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.