# Swap routing

Coffer's backend can suggest how to split an exact input amount among pools containing the same token pair. Each route is a direct swap between those two mints. This service does not construct a multi-hop route through an intermediate token or submit transactions.

The backend routing behavior described here comes from local `backend-v2` branch `v5.1`, revision `a886497`. SDK quote behavior comes from `@cubee_ee/sdk` 0.11.1, revision `09cc776`, for contracts `96a2ee2`. These revisions are not interchangeable; the checked router does not implement every rule in the current SDK quote path.

## Requesting a candidate allocation

```ts
const route = await backend.getSwapRoute(
  tokenInMint,
  tokenOutMint,
  amountInRawString,
  inputMintDecimals,
);
if (!route.ok) throw new Error(route.error.humanMessage);
```

`backend` is a `CubeBackendClient`, mint addresses are strings, and the input amount is a decimal string in raw input-token units. Passing actual input decimals matters for display and XP estimates. The checked REST route accepts `tokenIn`, `tokenOut`, `amountIn`, and `decimalsIn`; see the [API reference](api-reference.md#routing-estimates).

The optimizer begins with chunks of about one 128th of the requested input, chooses a candidate by marginal estimated output, and refines the allocation with smaller chunks. It also compares with single-pool alternatives. This is a heuristic search, not a proof of globally optimal execution. If it finds no candidate or cannot reconcile the allocated input to the full request, the service returns an empty route set and zero expected output.

The response includes raw allocated amounts and estimated outputs per pool, percentage allocations, token indices/programs, nullable vault addresses, and aggregate price/fee displays. An empty route is a failure to find an allocation, not permission to execute a partial amount silently.

## Limits of the checked backend implementation

The router reads pool data and uses `calcOutGivenIn`, but it does not call the full `CubicPoolClient.quoteSwap`. Its local implementation has important differences from the v5 quote:

- It derives an output-reserve cap by subtracting accrued protocol fees from actual balances. In v5, actual balances already exclude that separately tracked bucket.
- It uses indexed pool swap-fee metadata, including a fallback for zero-valued fee metadata, instead of consistently using the synced pool fee.
- It does not include the current sell-off window/cap and dynamic output-fee integration used by the deployed swap instruction.

Consequently, its estimated output and allocation can differ from executable v5 swaps. These are source findings at `a886497`; a later or different backend deployment may have addressed them. Updating the SDK dependency alone does not replace the router's surrounding reserve/fee/state logic.

The SDK method signature also accepts `slippageBps` and `pool`, and declares `minReceived` plus per-route `minAmountOut`. The checked backend DTO/controller neither uses those extra query options nor emits those minimum fields. Do not use the TypeScript type as evidence that an output floor was returned. In particular, do not interpret missing floor fields as zero.

## Turning an allocation into instructions

For each suggested pool, create a `CubicPoolClient`, sync it, find the mint indices from that synced state, and quote the allocated amount locally. Reject and recompute a route when any leg is disabled, unsupported, cap-limited, stale, or unquotable. A successful quote accounts for fees and state at the read time; it cannot reserve that state until execution.

```ts
import BN from "bn.js";
import { PublicKey } from "@solana/web3.js";
import { CubicPoolClient } from "@cubee_ee/sdk";

const client = new CubicPoolClient({
  config,
  poolAddress: new PublicKey(candidate.poolAddress),
});
const synced = await client.sync();
if (!synced.ok) throw new Error(synced.error.humanMessage);

const tokenInIndex = synced.data.tokens.findIndex(
  (t) => t.mint.toBase58() === tokenInMint,
);
const tokenOutIndex = synced.data.tokens.findIndex(
  (t) => t.mint.toBase58() === tokenOutMint,
);
const quote = client.quoteSwap(
  tokenInIndex,
  tokenOutIndex,
  new BN(candidate.amountIn),
  5_000, // 0.5% in SDK hundredths-bps units
);
if (!quote.ok) throw new Error(quote.error.humanMessage);

const built = client.buildSwapTx({
  user: walletPublicKey,
  tokenInIndex,
  tokenOutIndex,
  amountIn: quote.data.amountIn,
  minAmountOut: quote.data.minAmountOut,
});
if (!built.ok) throw new Error(built.error.humanMessage);
```

This example builds one unsigned leg from application-provided `candidate`, config, mints, and wallet. The SDK checks invalid indices, so a missing mint is an error. Sum the validated leg input amounts and require that they equal the user's chosen total. Display the sum of newly quoted net outputs and the sum of signed minimum outputs, rather than mixing old backend estimates with new SDK floors.

For multiple legs in distinct pools, compose the instructions only when they fit the message and compute limits. `compileBuiltTx` accepts one pool ALT; composing a multi-pool message may require your own v0 message compilation with all needed lookup tables. When legs are sent in separate transactions, partial completion is possible and the UI must account for confirmed legs before retrying the remainder.

For repeated swaps touching the same pool, independently quoting each against one unchanged cache does not simulate their sequence. Consolidate the leg where appropriate or simulate the cumulative state changes. STLD's internal sequence is handled by its dedicated [single-token quote](../sdk/single-token-deposit.md).

## Prices, fees, and XP

The SDK quote exposes net `amountOut`, gross output, dynamic output fee, input swap fee, and protocol share. `priceImpactHbps` uses hundredths of a basis point. The backend display uses percentage fields and a best-candidate spot baseline; those values are not the same unit or necessarily the same calculation. See [pool mathematics](../technical/math.md) for the transaction quote.

Backend `estimatedXp` is a rewards estimate derived from fee valuation, the epoch rate, and an optional authenticated referral boost. It is neither on-chain output nor a guaranteed immediate credit. Confirmed transactions still need to be indexed and processed by the [three-hour XP accrual](../rewards/cube-xp.md).

Sources: [routing implementation](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/swap/swap-router.service.ts),
[accepted query fields](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/pool/dto/swap-route-query.dto.ts),
[SDK pool quote/build path](https://github.com/coffer-so/sdk/blob/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/clients/CubicPoolClient.ts).
