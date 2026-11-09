> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Resource Locks

> LI.FI Intents use secure, flexible resource locks to enable trust-minimized cross-chain flows and new ways to design intent-based protocols.

Resource locks allow users to commit tokens to something or someone without making an on-chain transaction to issue the lock or recover the lock after its expiry. The tokens must stay out of reach for a set period — longer than a claim is expected to take to settle.

In a sense they work like a credit account: users can give protocols guaranteed allowances.

## Actors

In a resource lock flow, there are three actors:

1. Sponsor, user, owner: The actor that initially owns the funds in the resource lock.
2. Allocator, manager, credible commitment: The actor that validates that no conflicting locks are issued.
3. Arbiter, validator, settlement: The actor that can validate whether a lock has been resolved and can be paid.

Depending on the resource lock system, each user chooses an allocator, and each lock chooses the arbiter. When the term "the arbiter" is used, it refers specifically to the arbiter of a lock, not to a general single arbiter that the user has chosen.

Think of a lock as a time-bound approval, co-signed by an allocator, to a specific protocol (arbiter).

While resource lock flows can look very different depending on the application, they generally follow one setup step and four steps per lock:

**Step 1**

**Setup.** The **sponsor** makes a deposit into a resource lock, if one does not already exist. This happens once, not per lock.

**Step 2**

The **sponsor** signs a lock that describes a desired outcome.

**Step 3**

The **allocator** ensures that the appropriate funds exist for the lock to be valid. In other words, if the sponsor has deposited 10 tokens, no set of approved locks shall exceed 10 tokens. The allocator then co-signs the lock.

**Step 4**

The desired event takes place.

**Step 5**

The **arbiter** validates that the event has taken place and releases the token to the appropriate recipient.

In this flow, two signatures and one transaction – setting the deposit aside – are required. The lock has to be signed by the sponsor and co-signed by the allocator. Then the arbiter makes the final call on whether the desired event took place.

## Resource-Locked Intents

In an intent system, the third step – the execution of the desired event – is usually done by a fourth actor: the **solver**, also called the **relayer**.

Additionally, there are generally two ways to build intent systems around resource locks:

##### Intent system as *executor* (Tribunal)

In these flows, the intent system describes how the action takes place on the destination chain. However, the system does not itself validate that the resource lock was completed.

These systems instead rely on **allocators as arbiters** or resource lock support by **general message passing** (GMP) protocols.

The advantage of such a system is its simplicity. The third party (whether allocator or GMP) generally handles the entire validation/settlement pipeline, and thus solvers only have to concern themselves with filling orders.

##### Intent system as *arbiter* (OIF)

In these flows, the intent system is the arbiter and usually describes how the entire system is constructed: output settlement, validation, and input settlement. This makes the system more flexible and customizable.

While these systems may also rely on GMPs, they can also use other – potentially cheaper – validation layers like optimistic oracles, storage proofs, or secret reveals (like HTLCs).

The advantage here is, indirectly, the complexity: because the system defines more of the logic itself, intents can express more and be more specialized.

For more on the resource locks OIF supports, read the chapter on [Input Settlement Implementations](https://docs.li.fi/lifi-intents/architecture/settlement.md#input-settlement).

## Trust Assumptions

In a resource lock system, the actors need to trust each other, but generally, no single actor can independently access any funds.

* The sponsor needs to trust the arbiter such that **issued** locks are not fraudulently *finalized*. However, the arbiter can never access any of the sponsor's funds without a relevant signed lock. The trust assumptions between the sponsor and the arbiter are equivalent to permit/permit2 approvals.
* The arbiter/solver needs to trust that the allocator does not co-sign overlapping locks that exceed the tokens the user has deposited. Otherwise, a filled intent may not have enough tokens left to pay the solver for fulfilling the lock.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.