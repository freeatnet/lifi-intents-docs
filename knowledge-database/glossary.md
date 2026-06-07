> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Glossary

> Intent systems and cross-chain bridges use many specialized terms. This page explains arbiters, allocators, sponsors, locks, GMPs, settlers, and more.

## Arbiter

An entity, contract, or mechanism capable of determining when the desired action of a resource lock has been fulfilled. It may also act as an allocator.

## Allocator

An entity that validates that a lock does not exceed a user’s current balance. It may also act as an arbiter. When depositing into a lock, an allocator is chosen.

## emissary

For smart wallets unable to provide stable signatures, an emissary can be used to produce stable signatures. The emissary is usually a trusted entity, meaning that once a signature is issued it cannot be redacted. The terminology is specific to The Compact.

## Fill First

A swap flow where the tokens are delivered to the user *before* the input tokens are collected by a protocol. These flows use specialized wallet techniques like [resource locks](#resource-lock) to remain safe.

## Output Settler

A contract that accepts solver fills on destination chains and generates attestations used by oracle/input-settlement verification.

## GMP

Generalized Message Passing. Describes a method to send messages between two chains. Examples include Wormhole, LayerZero, Hyperlane, and more.

## Lock

An allowance issued by a user to an arbiter. Before locks are valid, they must be signed by the user and co-signed by the allocator.

## Input

The starting point. Input assets refer to the assets paid into the intent system. The input chain refers to the chain(s) of the input assets.

## Integrator

See [Intent Issuer](#intent-issuer).

## Intent

An issuance of a desired action. Intents are often used to describe swaps but can also describe desired cross-chain interactions. Unlike swaps, intents should generally be composable. Intents are usually self-contained, meaning they describe both the desired end state and the payment to achieve that state; the output and input, respectively.

## Intent Issuer

Someone who issues intents for a cross-chain intent system. The intent issuers specifies how the intent is configured. It is then up to [solvers](#solver) to determine whether or not they will fill the intent.

## Optimistic

A validation method that assumes statements are true unless disputed.

## Oracle

A contract that verifies whether solver delivery conditions were met so settlement can proceed. For same-chain intents, the output settler can be used directly as the oracle path. Oracles may use optimistic systems, messaging bridges, light clients, and other mechanisms.

## Order Server

A server that sits between intent issuers and solvers aiding with broadcasting and distribution of intents.

## Output

The endpoint. Output assets refer to the desired tokens to be paid to the user (or used within a larger action). The output chain refers to the chain(s) of the output assets.

## Resource Lock

An umbrella concept for lock-based input settlement mechanisms (for example escrow and Compact) that hold user funds until settlement conditions are met. You can find a primer in our [knowledge hub](/lifi-intents/knowledge-database/resource-locks).

## Input Settler

A contract on the origin chain that verifies proofs of delivery and releases locked funds to the solver. In a resource-lock-based system, the input settler often acts as an arbiter.

## Solver

A specialized third party who fulfills cross-chain intents using a variety of liquidity sources. They participate in the intent system with the goal of earning margins from the difference between the cost of achieving the end state and the inputs.

## Sponsor

The owner of the tokens to be used for locks in a resource lock.

## Validator Layer

An entity capable of validating whether certain information has occurred. Examples include GMP, optimistic proof, or bridges.
