> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Reputation

> The LI.FI order server tracks whether solvers fill the orders they win on time. Solvers that miss too many are paused for a while and come back through a recovery trial.

To prevent competing solvers from causing failed transactions and wasting gas, intent issuers can make their orders **exclusive** to one solver. When your quote wins, the order names your `exclusiveFor` address, and for a short period only you can fill it. You can read more about exclusive orders in the [auctions section](https://docs.li.fi/lifi-intents/for-solvers/auctions.md#exclusive-limit-order).

The order server checks whether you fill the orders built from your quotes on time, including orders that are not exclusive. If you miss too many, your solver is paused for one destination chain or for all chains. When the pause ends, you come back through a recovery trial.

You can check your status in the [Solver UI](https://intents.li.fi/) or through the [API](#checking-your-reputation).

***

## Solver Selection

When an intent issuer requests a quote, the order server evaluates all submitted solver quotes for the route. It leaves out quotes from solvers that are paused for that destination chain or for all chains. A solver on a [recovery trial](#recovery-trials) is also left out while it already has 3 trial orders waiting for a fill. The best price among the remaining quotes wins.

***

## Orders That Count

Your reputation is based on whether you fill two kinds of orders:

* Orders that name one of your [registered addresses](#registering-addresses) as their exclusive solver.
* Orders without an exclusive solver that the order server built from your quote.

Your **window** for an exclusive order ends at its exclusive deadline. For an order without an exclusive solver, it ends 90 seconds after the order server issued your quote when the order starts on an EVM chain, and 120 seconds after it on other chains. Either way, it ends at the fill deadline if that comes first.

You fill on time when you fill every output yourself by the end of the window. For an order without an exclusive solver, a fill from any of your registered addresses counts.

These results are a **missed deadline**:

* Another solver filled an output first on an exclusive order, or after your window ended.
* Nobody filled an output.

A refund also counts as a miss, unless you had already filled every output on time.

Some results count as neither a fill nor a miss:

* You filled every output yourself, but after the window ended.
* The order has no exclusive solver and another solver filled it first by the end of your window. Without an exclusive solver anyone may fill the order.

These orders do not end a run of misses in a row and do not count toward a recovery trial.

### Excluded Orders

These orders never count toward your reputation:

* An output includes a destination call, such as a swap.
* The order was opened after your quote expired.
* The order's input is below 98% or above 105% of your quoted input.
* The order was opened with too little time left in your window to fill it.

***

## Pauses

The order server checks two rules, for your solver as a whole and for each destination chain on its own. If either rule is met, your solver is paused for that chain, or for every chain:

| Rule | You are paused when |
| - | - |
| Recent misses | Among your counted orders opened in the last 48 hours, the misses are more than 12% of the orders, plus 2 |
| Misses in a row | You miss 20 counted orders in a row, however long ago they were opened |

The [example](#example) below shows how this plays out.

### What a Pause Does

While a destination chain is paused, the order server does not select your quotes for orders to that chain. While your whole solver is paused, it does not select your quotes for any chain. You can keep submitting quotes during a pause.

Keep filling the orders you won before the pause, because they still count toward your reputation.

### Pause Length

A pause lasts 1 hour, and doubles for each earlier pause in the last 28 days, up to 32 hours. A chain pause only counts earlier pauses of that chain, and a whole-solver pause only counts earlier whole-solver pauses.

When a pause ends, its counts start again from zero, so the misses that caused it cannot pause you a second time.

> [!NOTE]
>
> The numbers on this page are the current settings and can change. The [reliability endpoint](#checking-your-reputation) shows the limits that apply to you.

***

## Recovery Trials

When a pause ends, a **recovery trial** starts for the same chain, or for the whole solver. The order server selects your quotes again, but with a limit:

* It stops selecting your quotes while 3 of your trial orders are waiting for a fill. A slot frees up when an order is filled or its window ends.
* The trial passes after 20 on-time fills, once all trial orders have an outcome and none of them is a miss.
* A missed trial order fails the trial and starts a new pause.

***

## Example

Say your solver fills orders to Arbitrum and Base, and it has not been paused in the last 28 days. In the last 48 hours you have 50 counted orders to Arbitrum and 150 to Base. Then your Arbitrum node falls behind, and some of those Arbitrum orders start to miss.

**1. Misses 1 to 8**

Nothing happens yet. With 50 counted orders, Arbitrum allows up to 8 misses: 12% of 50 is 6, plus 2.

**2. Miss 9: Arbitrum is paused**

The 9th miss puts Arbitrum over its limit, so Arbitrum is paused for 1 hour and you get an email. The order server stops selecting your Arbitrum quotes, but your Base quotes can still win.

Across both chains, you have 9 misses in 200 counted orders. Your whole solver allows up to 26 misses (12% of 200 is 24, plus 2), so it is not paused on other chains.

Keep filling any Arbitrum orders you won before the pause, because they still count toward your reputation.

**3. After 1 hour: recovery trial**

The Arbitrum pause ends and a recovery trial starts, with the Arbitrum counts starting again from zero. Your Arbitrum quotes can win again, but the order server stops selecting them while 3 trial orders are waiting for a fill.

**4. 20 on-time fills: back to normal**

Once you reach 20 on-time fills, and no trial order has missed or is still waiting for an outcome, the trial passes and Arbitrum is back to normal.

If you miss a trial order instead, Arbitrum is paused again, this time for 2 hours, because it is the second Arbitrum pause in 28 days. Another trial starts when that pause ends.

***

## Checking Your Reputation

The Reputation page in the [Solver UI](https://intents.li.fi/) shows your status for each destination chain, your recent misses and your pause history. The same data is available through the API:

| Method | Path | Returns |
| - | - | - |
| `GET` | [`/api/v1/solver/reliability`](https://order.li.fi/docs#tag/solver-api-v1/GET/api/v1/solver/reliability) | Your current status, active pauses and trials, and miss counts |
| `GET` | [`/api/v1/solver/reliability/pauses`](https://order.li.fi/docs#tag/solver-api-v1/GET/api/v1/solver/reliability/pauses) | Your pause history, newest first |

```bash cURL theme={"system"}
curl 'https://order.li.fi/api/v1/solver/reliability' \
  -H "x-api-key"':'"${SOLVER_API_KEY}"
```

```ts TypeScript theme={"system"}
const response = await fetch('https://order.li.fi/api/v1/solver/reliability', {
  headers: { 'x-api-key': process.env.SOLVER_API_KEY! },
});
const reliability = await response.json();
```

See the [API reference](https://order.li.fi/docs#tag/solver-api-v1) for every field. The 28-day miss rate in `longTermMisses` is there for your own tracking and never pauses you.

***

## Pause Emails

When your solver is paused, the order server emails the address of your Solver UI account. Make sure that address is confirmed in the Solver UI.

If you keep failing recovery trials, you get at most one email a day about them.

***

## Solver Setup

To take part, sign up for the Solver UI:

* **Production**: [https://intents.li.fi/](https://intents.li.fi/)
* **Testnet**: [https://devintents.li.fi/](https://devintents.li.fi/)

After signing up, set your **solver name** and create an **API key**. Pass your key in the `x-api-key` header on all solver endpoints. See [Authentication](https://docs.li.fi/lifi-intents/authentication.md#solvers-api-key-required) for details.

### Registering Addresses

To register solver accounts, the solver signs a server-issued message to prove ownership and submits the pairing to the Order Server. The same flow serves EVM and Solana accounts: fetch a one-time message from `GET /api/v1/solver/register/message`, sign it — **ECDSA** or **EIP-1271** on EVM, **ed25519** on Solana — and submit it to [`POST /api/v1/solver/register`](https://order.li.fi/docs#tag/solver-api-v1/post/api/v1/solver/register).

See [Registering Accounts](https://docs.li.fi/lifi-intents/for-solvers/quoting.md#registering-accounts) for the step-by-step walkthrough with code.

A successful registration returns `{ "success": true }`.

#### Possible Errors

* `400 Bad Request`: Invalid request data
* `401 Unauthorized`: Invalid signature or API key
* `403 Forbidden`: Address already registered

The registration message is single-use and consumed on the first submission attempt — if registration fails, fetch a fresh message before retrying.

> [!NOTE]
>
> Each address can be registered to only one API key, but a single API key can have many addresses registered.


This documentation is built and hosted on [Mintlify](https://mintlify.com), a developer documentation platform.