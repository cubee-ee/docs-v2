# Pricing Model

Cube implements a **weighted constant-product** AMM with virtual liquidity.
Each pool has 2–10 tokens with configurable weights; swap pricing is
deterministic on-chain. Integrators do not need to re-implement the
math — quotes are available via the on-chain program or the public
swap router (see [Swap Routing](../integration/swap-routing.md)).

## Swap formula (high-level)

```
amountOut = balanceOut * (1 - (balanceIn / (balanceIn + amountInAfterFee)) ^ (weightIn / weightOut))
```

Properties an integrator should know:

- **EXACT_IN only.** The swap instruction takes `amount_in` and
  `minimum_amount_out`; the program computes the exact output.
- **Asymmetric weights.** A 50/50 pool prices symmetrically; an 80/20
  pool moves slower against the heavier token.
- **Output cap.** The computed output cannot exceed the
  LP-accessible balance of the output token. The transaction reverts
  with `AmountOutExceedsBalance` if it would.
- **Rounding is conservative.** Rounding direction is always chosen
  so the protocol never overpays the user.

For the off-chain quote helper used by aggregators see
[`@cube/sdk` → `CubicPoolClient.getSwapQuote`](../sdk/index.md).

## Fees

- **Swap fee** is deducted from `amount_in` before the swap formula
  runs. Encoded as a `u32` in hundredths of a basis point — see
  [Pool Parameters](../overview/pool-parameters.md) for the exact range.
- **Protocol fee** is a share of the swap fee carved out for the
  treasury. Tracked separately on-chain; collected by the protocol
  admin. LPs see the remainder accumulate in the pool's actual
  balances over time.

## Spot price

```
spotPrice = (balanceIn / weightIn) / (balanceOut / weightOut)
```

This is the instantaneous price for an infinitesimally small trade
and is **not** what a real trade clears at — actual trades experience
price impact proportional to size relative to pool depth.
