# Aqua0 RFQ

Aqua0's backend chooses the liquidity to add just before a Uniswap V4 swap. It uses its existing
fleet planner and Rust-backed V4 simulator to quote that liquidity. Tycho does not independently
choose the JIT amounts or infer off-chain commitments from pool events.

The integration has two steps:

1. Read indicative levels from Aqua0. This does not reserve money or produce a signature.
2. When encoding a selected route, request an exact-input approval. Aqua0 checks current backing,
   reserves it, and returns short-lived signed `hookData`. The existing Tycho Router V3 and
   Uniswap V4 executor execute it. An approval can be refused even if an earlier preview succeeded.

`rfq:aqua0` is an RFQ component, not a separately indexed on-chain AMM.

## Register a market

Each client represents one `(chain, pool, class)` market. Obtain enabled markets and token metadata
from Aqua0. Registration here does not automatically make another solver use the protocol.

```rust,ignore
use tycho_common::models::Chain;
use tycho_simulation::rfq::{
    protocols::aqua0::{
        client_builder::Aqua0ClientBuilder, models::Aqua0Market, state::Aqua0State,
    },
    stream::RFQStreamBuilder,
};

let market = Aqua0Market {
    pool_id, // bytes32 pool ID from Aqua0
    class_id, // decimal strategy class ID from Aqua0
    amount0_samples: vec!["1000000000000000".into(), "10000000000000000".into()],
    amount1_samples: vec!["1000000".into(), "10000000".into()],
};
let aqua0 = Aqua0ClientBuilder::new(Chain::Base, rfq_base_url, market).build()?;
let stream = RFQStreamBuilder::new()
    .set_tokens(token_metadata)
    .await
    .add_client::<Aqua0State>("aqua0", Box::new(aqua0));
```

The base URL ends in `/api/tycho/rfq`. Samples are in the respective token's smallest units. Select
sizes appropriate to its decimals and the market. Quotes and swap approvals do not require Aqua0
operator or LP beta credentials. The backend can temporarily refuse callers who leave approvals
unused; callers must handle errors and `429` responses, not retry in a tight loop.

## API and identity

- `GET /state`: query `chainId`, `poolId`, `classId`, `amount0Samples`, `amount1Samples`. Amount
  samples are comma-separated decimal integers. Returns expiring levels in both directions,
  selected ranges, and a state version. Existing holds are subtracted by the backend's shared
  quote service exactly once. Reading state never creates another hold.
- `POST /quote`: send the exact amount, tokens, component ID, pool, class, request ID and expected
  router. Returns the signed hook data, actual router, approval nonce and deadline.

The component ID is the Keccak-256 hash of UTF-8
`aqua0-rfq-v1:{chainId}:{lowercasePoolId}:{decimalClassId}`. The client derives it locally and checks
backend responses against it, so encoding needs one quote POST, not an extra state GET first.

The client shares an HTTP connection pool. Each request has a timeout covering response-body
reading as well as headers. State and quote identity, exact input and expiry are checked before
accepting a response. The solver should still simulate the complete encoded transaction and check
its minimum output before broadcasting. Indicative interpolation is not a guarantee of settlement.

## Execution

Supported execution chains are Base, Arbitrum, Polygon and Robinhood. This describes available
Tycho execution support, not UniswapX order availability or Aqua0 market readiness on every chain.

The Aqua0 encoder uses the configured `uniswap_v4` executor as an alias. It does not add an Aqua0
address to `executor_addresses.json` or require a new Solidity executor. Router V3 is Tycho's
router version; V4 refers to Uniswap V4.

Use Tycho's current router registry and confirm the same router is configured and allowlisted by
Aqua0. Old Aqua0 deployments were pinned to superseded routers. An existing `UniswapXFiller`
cannot be retargeted: its router and reactor are immutable. Migrating it requires deploying and
configuring a replacement. This RFQ plugin also works for solvers using their own settlement
integration; running Aqua0's UniswapX worker is not required to request quotes.

Current scope:

- Exact input, same chain, no split routes.
- Exactly one Aqua0 leg, first or only hop, whose input equals the whole solution's input.
- Inputs above the largest advertised sample and expired state are refused.

The whole solution is checked before requesting approvals. A group-local first-hop check alone
would miss Aqua0 appearing after a different protocol. The encoder marks itself as blocking on a
quote so it follows Tycho's RFQ scheduling path, then delegates byte packing to
`UniswapV4SwapEncoder`.

## Verification

```bash
cargo test -p tycho-simulation rfq::protocols::aqua0 --lib
cargo test -p tycho-execution aqua0 --lib
cargo test -p tycho-execution --bin aqua0-uniswapx-encode
```

The included `Aqua0TychoBaseFork.t.sol` exercises reactor/filler/router/executor plumbing on forks
using seeded pools without an Aqua0 hook. It is not proof of an end-to-end Aqua0 fill. Aqua0's
separate contracts repository has a Base proof with locally deployed JIT contracts. A release still
needs a complete fork fill using the real backend approval, selected ranges, deployed hook and
settlement contracts, followed by an explicitly approved small live fill.
