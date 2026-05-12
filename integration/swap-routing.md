# Swap Routing

The Cube backend includes a smart order router that splits swaps across multiple pools for optimal execution. This page explains how the routing algorithm works and how integrators can use it.

---

## Overview

When a token pair exists in multiple pools, routing all volume through a
single pool can cause unnecessarily high price impact. The Cube router splits
the input across eligible pools using the same BigInt fixed-point math that
the SDK and on-chain contract use for final quotes.

The key rule: a route is only returned if every split can be executed by the
contract against LP-accessible liquidity. The router never silently caps
outputs at vault balances.

---

## Algorithm

### 1. Pool Discovery

The router queries the database for all enabled pools containing the input token, then filters for pools that also contain the output token.

### 2. On-Chain Balance Fetch

For each candidate pool, the router fetches **fresh on-chain balances** via RPC (with a 10-second timeout). If on-chain data is unavailable for a pool, that pool is skipped entirely — stale database values are not used for routing.

### 3. Eligibility

Each candidate pool must:

- be enabled and have swaps enabled
- have fresh on-chain state
- contain both tokens
- produce a non-zero fee when `swap_fee_rate > 0`
- have enough LP-accessible output liquidity

### 4. BigInt Allocation

The optimizer uses integer amounts and exact SDK quotes:

1. Start with zero allocation for each eligible pool
2. Pick a chunk size based on total input
3. For each chunk, quote `currentAllocation + chunk` through each pool with
   `@cube/sdk` `calcOutGivenIn`
4. Assign the chunk to the pool with the best marginal output
5. Reduce chunk size until the full input is allocated exactly

This is slower than a floating-point derivative search, but it avoids
precision drift on large balances and exactly matches the contract's reject
conditions.

### 5. Output Calculation

Once allocations are determined, each pool's expected output is computed using the on-chain swap formula:

```
amountOut = virtualBalanceOut * (1 - (virtualBalanceIn / (virtualBalanceIn + amountInAfterFee)) ^ (weightIn / weightOut))
```

Inputs match cubic-pool's on-chain `swap` exactly: raw
`virtualBalanceIn` / `virtualBalanceOut` drive the formula and the
LP-accessible output balance is supplied only as the cap:

```
lpActualOut = actualBalanceOut - protocolFeesOwedOut    // saturating
```

If the SDK math reports `AmountOutExceedsBalance` (i.e. the formula's
`amountOut` would exceed `lpActualOut`), the candidate route is treated as
unroutable; output is not silently capped.

### 6. Response

The router returns:
- Per-pool splits with amounts, percentages, and vault addresses
- Per-leg token program IDs for mixed SPL Token / Token-2022 pools
- Total expected output
- Effective price (output per input)
- Price impact (vs. spot price)
- Partial liquidity flag if insufficient capacity

---

## Fee Math

The router replicates the exact on-chain fee calculation:

```javascript
fee = floor(amountIn * feeRate / 1_000_000)
amountInAfterFee = amountIn - fee
```

Swaps where the fee rounds to zero (when `feeRate > 0`) are rejected, matching the on-chain behavior.

---

## Spot Price

Per-pool spot price (no-trade reference):

```
spotPrice = (virtualBalanceOut / weightOut) / (virtualBalanceIn / weightIn)
```

The response's `spotPrice` is a weighted average across all routed pools.

---

## Price Impact

```
priceImpact = (spotPrice - effectivePrice) / spotPrice * 100
```

A positive value means the trade executes worse than spot. Larger trades have higher impact.

---

## Integration Example

### Step 1: Query the Route

```bash
curl "https://api.cubee.ee/api/pools/swap-route?tokenIn=So11111111111111111111111111111111111111112&tokenOut=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v&amountIn=1000000000"
```

### Step 2: Build Transactions

For each route in the response, build a Solana `swap` instruction:

```typescript
for (const route of response.data.routes) {
  const ix = await program.methods
    .swap(
      new BN(route.amountIn),
      new BN(minAmountOut),     // apply your slippage tolerance
      route.tokenInIndex,
      route.tokenOutIndex,
    )
    .accounts({
      pool: new PublicKey(route.poolAddress),
      tokenMintIn: new PublicKey(tokenInMint),
      tokenMintOut: new PublicKey(tokenOutMint),
      userTokenAccountIn: userTokenAccountIn,
      userTokenAccountOut: userTokenAccountOut,
      vaultIn: new PublicKey(route.vaultIn),
      vaultOut: new PublicKey(route.vaultOut),
      user: wallet.publicKey,
      tokenProgramIn: new PublicKey(route.tokenProgramIn),
      tokenProgramOut: new PublicKey(route.tokenProgramOut),
    })
    .instruction();

  transaction.add(ix);
}
```

### Step 3: Sign and Send

```typescript
const signature = await sendTransaction(transaction, connection);
```

If the route spans multiple pools, each swap instruction is independent and can be included in the same transaction (Solana supports multiple instructions per transaction).

---

## Accuracy

The router delegates every candidate and final output calculation to
`@cube/sdk`, which is the TypeScript fixed-point port of the Rust
`CubicMath::calc_out_given_in` implementation. It subtracts
`protocolFeesOwed` before quoting and treats SDK `AmountOutExceedsBalance`
errors as unroutable pools.

Always apply a slippage tolerance (e.g., 0.5%–1%) to `minimum_amount_out` to account for:
- Price movement between quote and execution
- Minor precision differences
- Other trades executing before yours
