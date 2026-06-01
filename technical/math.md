# Pricing Model

Cube implements a **weighted constant-product** AMM with virtual liquidity.
Each pool has 2–9 tokens (the on-chain program supports up to 10, but
the UI/SDK deploy flow caps at 9 because `initialize_cubic_pool` for
N=10 overflows the 1232-byte legacy-tx limit) with configurable
weights; swap pricing is deterministic on-chain. Integrators do not
need to re-implement the math — quotes are available via the on-chain
program or the public swap router (see
[Swap Routing](../integration/swap-routing.md)).

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

The instantaneous price (limit as trade size → 0) uses **virtual
balances** weighted by the per-token weights:

```
spotPrice(in → out) = (vbIn / wIn) / (vbOut / wOut)
                    × 10^(decimalsOut − decimalsIn)
```

Where:
- `vbIn`, `vbOut` are the **virtual** balances (raw on-chain `u64` —
  not actual vault balances; see [Pool Parameters](../overview/pool-parameters.md#virtual-balance-vs-actual-balance)).
- `wIn`, `wOut` are the normalised weights in basis points (sum across
  active slots == 10 000).
- The decimal correction puts the result in **units of `out` per 1 unit
  of `in`** in human-readable terms.

The reference implementation is `WeightedMath::calc_spot_price` in
`programs/cubic-pool/src/math` — it mirrors Balancer's classic
weighted spot-price formula. Using the simpler `vbIn / vbOut` ratio
**without weights** is incorrect for non-50/50 pools and will be off
by a factor of `(wOut / wIn)`.

This is the price for an infinitesimally small trade — actual trades
experience price impact proportional to size relative to pool depth.
