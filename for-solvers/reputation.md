> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Reputation

> The LI.FI Order Server tracks solver performance and assigns each solver a score. This score helps improve user experience through better solver selection.

To prevent competing solvers from causing failed transactions and wasting gas, the Order Server allows intent issuers to specify **solver exclusivity** for their issued intents.\
This process is handled through solver selection and quoting. You can read more about quoting in the [quoting section](/lifi-intents/for-solvers/quoting).

To access the reputation system and monitor solver performance, the first step is to sign up for the Solver UI:

* **Production**: [https://intents.li.fi/](https://intents.li.fi/)
* **Testnet**: [https://devintents.li.fi/](https://devintents.li.fi/)

***

### Solver Setup

After signing up on the dashboard, users must configure their **solver identity** by setting a solver name.\
This identity will be used to create API keys and participate in the solver marketplace.

***

### API Keys

Once a solver identity is established, users can create and manage **API keys**. Pass your key via the `api-key` header on all solver endpoints. See [Authentication](/lifi-intents/authentication#solvers-api-key-required) for details.

API keys are required for:

* **Pushing Quotes**: Submit inventory quotes to the Order Server using the [`/quotes/submit`](https://order.li.fi/docs#tag/solver-api/post/quotes/submit) endpoint.
* **Registering Solver Accounts**: Registering solver accounts is required for reputation tracking.

***

### Registering Addresses

To register solver accounts, the solver must sign a message to prove ownership and then submit this pairing to the Order Server.\
The Order Server supports both **ECDSA** and **EIP-1271** signatures.

Submit the signed message, signature, and address to:\
[`/solver-api/account/register`](https://order.li.fi/docs#tag/solver-api/post/solver-api/account/register)

#### Example Response

```json theme={"system"}
{
  "id": 1,
  "address": "0x1234567890123456789012345678901234567890",
  "solverId": 5,
  "createdAt": "2024-01-15T10:30:00.000Z",
  "updatedAt": "2024-01-15T10:30:00.000Z"
}
```

#### Possible Errors

* `400 Bad Request`: Invalid request data
* `401 Unauthorized`: Invalid signature or API key
* `403 Forbidden`: Address already registered
* `404 Not Found`: Solver not found

<Note>
  Each address can only be registered to one API key. However, a single API key can have multiple addresses registered.
</Note>

***

### Performance Monitoring

The dashboard provides key metrics, including:

* Orders quoted
* Quotes submitted
* Orders filled
* Current reputation score

***

## Solver Selection

Solvers submit quotes to the Order Server containing details such as price, route, expiry, and their desired `exclusiveFor` value.

When an intent issuer requests a quote, the Order Server evaluates all submitted solver quotes and selects the best one(s).
The final selection is based on a combination of **price** and **reputation**, ensuring that the chosen solver aligns with the `exclusiveFor` parameter of the returned quote.

***

### Reputation

A solver’s reputation reflects its **reliability** and **fill performance**:

* **Reliability**: How often does the solver successfully fill a quote once assigned?
* **Fill Performance**: How quickly does the solver complete fills compared to others?

Price competition is evaluated separately from reputation, allowing both quality and efficiency to influence solver selection.
