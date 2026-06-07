> ## Documentation Index
> Fetch the complete documentation index at: https://docs.li.fi/llms.txt
> Use this file to discover all available pages before exploring further.

# Intent / Solver Marketplace

> LI.FI's intent marketplace for cross-chain and same-chain transfers, where the order server matches user intents against solver standing quotes.

export const IntentsSupportedRoutes = () => {
  const [routeSummary, setRouteSummary] = useState(null);
  const [error, setError] = useState(null);
  useEffect(() => {
    const fetchRoutes = async () => {
      try {
        const response = await fetch('https://order.li.fi/routes');
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data = await response.json();
        const routes = data.routes || [];
        const testnetNames = new Set(['Base Sepolia', 'Arbitrum Sepolia', 'Arc Testnet']);
        const mainnetRoutes = routes.filter(r => !testnetNames.has(r.fromChain.name) && !testnetNames.has(r.toChain.name));
        const pairMap = {};
        for (const r of mainnetRoutes) {
          const key = `${r.fromChain.name} → ${r.toChain.name}`;
          if (!pairMap[key]) {
            pairMap[key] = {
              from: r.fromChain.name,
              to: r.toChain.name,
              count: 0
            };
          }
          pairMap[key].count++;
        }
        const pairs = Object.values(pairMap).sort((a, b) => {
          if (a.from === a.to && b.from !== b.to) return -1;
          if (a.from !== a.to && b.from === b.to) return 1;
          return b.count - a.count;
        });
        setRouteSummary({
          pairs,
          total: mainnetRoutes.length
        });
      } catch (err) {
        setError(err.message);
      }
    };
    fetchRoutes();
  }, []);
  if (error) return <div>Error loading routes: {error}</div>;
  if (!routeSummary) return <div>Loading supported routes...</div>;
  return <div>
      <p>
        <strong>{routeSummary.total}</strong> active routes across{' '}
        <strong>{routeSummary.pairs.length}</strong> chain pairs.
      </p>
      <table>
        <thead>
          <tr>
            <th className="text-left"><strong>From</strong></th>
            <th className="text-left"><strong>To</strong></th>
            <th className="text-left"><strong>Active Routes</strong></th>
          </tr>
        </thead>
        <tbody>
          {routeSummary.pairs.map(pair => <tr key={`${pair.from}-${pair.to}`}>
              <td>{pair.from}</td>
              <td>{pair.to}</td>
              <td>{pair.count}</td>
            </tr>)}
        </tbody>
      </table>
    </div>;
};

export const IntentsSupportedChains = () => {
  const [chains, setChains] = useState(null);
  const [error, setError] = useState(null);
  useEffect(() => {
    const fetchChains = async () => {
      try {
        const response = await fetch('https://order.li.fi/chains/supported');
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data = await response.json();
        setChains(data);
      } catch (err) {
        setError(err.message);
      }
    };
    fetchChains();
  }, []);
  if (error) return <div>Error loading chains: {error}</div>;
  if (!chains) return <div>Loading supported chains...</div>;
  return <table>
      <thead>
        <tr>
          <th className="text-left"><strong>Chain</strong></th>
          <th className="text-left"><strong>Chain ID</strong></th>
          <th className="text-left"><strong>Type</strong></th>
        </tr>
      </thead>
      <tbody>
        {chains.map(chain => <tr key={chain.chainId}>
            <td><strong>{chain.name}</strong></td>
            <td><code>{chain.chainId}</code></td>
            <td>{chain.chainType}</td>
          </tr>)}
      </tbody>
    </table>;
};

LI.FI Intent / Solver Marketplace helps wallets and applications execute same-chain and cross-chain token transfers through a solver network. Solvers publish standing quotes (price curves for routes and amount ranges), and the order server matches intents to available solver inventory.

<Note>
  LI.FI Intent/Solver Marketplace is the official foundation of the [Open Intents Framework (OIF)](https://openintents.xyz/), an open-source initiative led by the Ethereum Foundation for permissionless interoperability.
</Note>

***

## What You Get as an Integrator

* **Competitive pricing.** Intents are matched against solver standing quotes and routed to the best available fill path.
* **Fast settlement.** Solvers use their own capital to deliver assets on the destination chain before settlement completes.
* **Same-chain and cross-chain support.** Route availability updates continuously based on active solver inventory.
* **Optional gasless order submission.** Off-chain order submission is gasless, while funding actions (approval/deposit into Escrow or Compact) are still on-chain transactions.
* **Calls on delivery.** Attach calldata to intents for custom logic after asset delivery.

***

## How It Works

1. **User expresses an intent.** The user deposits tokens into an input settler (escrow or Compact resource lock), then defines the desired output outcome without specifying execution details.
2. **Intent is distributed.** The lock generates a proof of deposit, which is broadcast to the order server and solver network.
3. **Intent is matched.** The order server matches the intent against existing solver quotes (standing inventory), not live per-intent RFQ bidding.
4. **Solver delivers.** The winning solver delivers the requested assets on the destination chain.
5. **Verification and settlement.** The output settler records the fill and generates attestations; the oracle/input settler verifies delivery and releases locked funds.

<img src="https://mintcdn.com/lifi/pOTDBSGpduSCqu2-/images/lifi-intent-small.png?fit=max&auto=format&n=pOTDBSGpduSCqu2-&q=85&s=add276a85219988e9d26317552afb8ef" alt="LI.FI Intent Lifecycle Diagram" width="2400" height="1488" data-path="images/lifi-intent-small.png" />

***

## Supported Chains

<IntentsSupportedChains />

Same-chain intents are live across all supported chains. Coverage includes major pairs and a broad long-tail token set (including Ondo assets). For the most up-to-date chain support, call [`GET /chains/supported`](https://order.li.fi/docs#tag/bridge-api/GET/chains/supported). Testnet chains are also available on the same endpoint.

## Supported Routes

<IntentsSupportedRoutes />

Route availability updates continuously based on active solver quoting. For the full route list including token-level detail, call [`GET /routes`](https://order.li.fi/docs#tag/bridge-api/GET/routes).

***

## Authentication

Integrator endpoints are fully open with no API key or rate limits. Solvers must register for an API key via the solver UI. See [Authentication](/lifi-intents/authentication) for details.

***

## Key Concepts

| Concept                           | Description                                                                                                                                                                                                                  |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Intent**                        | A user request that expresses the desired outcome without specifying execution details                                                                                                                                       |
| **Solver**                        | An entity that publishes standing quotes and fulfills intents using its own capital and liquidity                                                                                                                            |
| **Input Settler**                 | Umbrella term for source-side settlement contracts that release locked funds after verified delivery                                                                                                                         |
| **Escrow (InputSettlerEscrow)**   | Per-intent lock model where user funds are escrowed per order                                                                                                                                                                |
| **Compact (InputSettlerCompact)** | Deposit-based model where users can fund once and issue multiple intents from allocated balances                                                                                                                             |
| **Order Server**                  | Off-chain matching layer that maps intents to available solver quotes and distributes order flow                                                                                                                             |
| **Output Settler**                | Destination-side contract that accepts solver fills and generates attestations used for verification + settlement                                                                                                            |
| **Oracle**                        | Verification layer that proves delivery and enables settlement on the source chain. For same-chain intents, the output settler itself (OutputSettlerSimple) acts as the oracle. No cross-chain proof transmission is needed. |
| **StandardOrder**                 | Canonical on-chain order struct used for intents                                                                                                                                                                             |
| **`fillDeadline` vs `expires`**   | `fillDeadline` is solver fill cutoff; `expires` is final deadline after which refunds can be processed                                                                                                                       |
| **Order types**                   | Supported order types include limit, exclusive limit, Dutch auction, and exclusive Dutch auction                                                                                                                             |

For full definitions, see the [Glossary](/lifi-intents/knowledge-database/glossary).

***

## Next Steps

<CardGroup cols={2}>
  <Card title="Quickstart" icon="rocket" href="/lifi-intents/quickstart">
    Request a quote, submit an order, and track it in under 5 minutes
  </Card>

  <Card title="Intents API Overview" icon="code" href="/lifi-intents/intents-api/api-overview">
    Endpoints, base URLs, authentication, and order lifecycle
  </Card>

  <Card title="Solver API" icon="gears" href="/lifi-intents/for-solvers/api-overview">
    Solver endpoints, quoting, and filling orders
  </Card>

  <Card title="System Architecture" icon="diagram-project" href="/lifi-intents/architecture/overview">
    Settlers, oracles, and contract addresses
  </Card>
</CardGroup>
