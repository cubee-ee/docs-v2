# Pricing and Liquidity Math

The pool uses a weighted constant-product AMM with virtual balances. This page specifies the implementation in contracts `audit-fixes-excluded-SF` at `96a2ee2` and its SDK `0.11.1` counterpart at `09cc776`. Amounts in formulas are raw integer token units unless a human-unit price is explicitly shown.

## State and numerical scales

| Value | Meaning / scale |
|---|---|
| `actual_balance` | LP-owned token reserve; already excludes reserved protocol fees |
| `virtual_balance` | Reserve used by the pricing curve; it is not an additional token holding |
| `protocol_fees_owed` | Separate protocol claim held in the same vault |
| Token amounts and balances | `u64`, with the mint's decimals; SDK high-level amounts are `BN`, pure math uses `bigint` |
| Weights | `10,000 = 100%`; each weight is `100…9,900`, and all token-slot weights sum to `10,000` |
| Base swap fee | Denominator `1,000,000`; `3,000 = 0.3%`; maximum `100,000 = 10%` |
| Protocol share of base fee | Denominator `10,000`; `2,000 = 20%`; maximum `5,000 = 50%` |
| Selloff percentage, surge threshold and rate points | Denominator `10,000` |
| Surge kink | Whole percentage points: `90 = 90%` |
| Fixed-point intermediates | `ONE = 10^18` |

Pools have 2–10 token slots, with mint decimals at most 18. Transaction construction may impose additional message-size/account constraints; see [SDK](../sdk/index.md). A token can have a nonzero virtual balance while its actual reserve is zero. Such a sidelined token cannot fund a positive output until it receives actual liquidity.

For normal accounted transfers, the vault token amount equals `actual_balance + protocol_fees_owed`. Direct token donations to a vault can leave excess tokens outside those tracked amounts. All pricing and proportional liquidity calculations use the stored state, not the entire observed vault amount. Never subtract protocol fees from stored actual balances a second time.

## Weighted exact-input swap

Using input and output virtual balances `V_in`, `V_out`, weights `w_in`, `w_out`, and fee-adjusted input `x`, the real-number formula is:

```text
Y = V_out × [1 − (V_in / (V_in + x))^(w_in / w_out)]
```

Here `Y` is gross curve output before the dynamic fee. The contract implements it in this exact order:

```text
base_fp     = ceil(V_in × ONE / (V_in + x))
exponent_fp = floor(weight_in_fp × ONE / weight_out_fp)
power_fp    = min(pow_fp(base_fp, exponent_fp) + 1, ONE)
Y           = floor(V_out × (ONE − power_fp) / ONE)

require Y <= actual_balance_out
```

`weight_fp = weight_bps × ONE / 10,000`. The final `+1` is one fixed-point unit, not one output token unit. It is part of the implementation and must be preserved in a port. A computed output above the actual output reserve **reverts** with `AmountOutExceedsBalance`; it is not silently capped.

The input ratio contains quantities in the same token units, so decimals cancel there. The output multiplication is already in output-token raw units. No cross-token decimal scaling is required inside `calc_out_given_in`.

The pool's stored input reserve grows by gross input minus the protocol's base-fee share, not just by `x`: the remaining base fee becomes LP reserves. Therefore the post-trade invariant is not a fee-free constant. Virtual and actual reserves move together by the same raw amounts on each swap; their ratio need not remain constant across trading.

### Fixed-point log, exponential and power

The SDK mirrors the deployed `LogExpMath` rather than JavaScript `Math.pow`:

- `pow_fp(x, y) = exp_fp(y × ln_fp(x))`, with integer operation order preserved.
- Log and exponential use a ten-step constant decomposition.
- The logarithm's residual uses six odd-series terms; the exponential residual uses twelve Taylor terms.
- Intermediate exponent arguments must lie in `[-41, 46] × ONE`; out-of-range values fail rather than saturating to a guessed quote.
- Wide multiply/divide operations use a 256-bit intermediate in Rust; checked storage and ordinary fixed-point operations still enforce their `u64`/`u128` bounds.
- Special power cases include `pow(x, 0) = ONE`, `pow(0, y) = 0` for nonzero `y`, and `pow(ONE, y) = ONE`.

This is bounded integer approximation of transcendental math. Replacing it with floating point, a different log/exp approximation, or algebraically reordered integer divisions can change output. Use `calcOutGivenIn` for the curve or `CubicPoolClient.quoteSwap` for the complete fee/window-aware quote.

## Fees and balance updates

Let gross input be `A`, base fee rate `s`, protocol share rate `p`, and output surge charge `S`:

```text
base_fee       = ceil(A × s / 1,000,000)
protocol_fee   = ceil(base_fee × p / 10,000)
x              = A − base_fee
Y              = curve_output(x)
user_output    = Y − S

input_actual_new   = input_actual + A − protocol_fee
input_virtual_new  = input_virtual + A − protocol_fee
input_fees_new     = input_fees + protocol_fee
output_actual_new  = output_actual − Y
output_virtual_new = output_virtual − Y
output_fees_new    = output_fees + S
```

The base fee and protocol share both round **up**. The input-token LP fee income is `base_fee − protocol_fee`. For a dust-sized one-unit base fee, the rounded protocol share can equal the whole fee even when the configured share is below 100%; it never exceeds it. A zero rate produces a zero fee.

Example: 3,998 raw input units at rate `1,000` (0.1%) owe `ceil(3.998) = 4` raw base-fee units. With a 20% protocol share, `ceil(4 × 2,000 / 10,000) = 1` goes to the protocol and 3 to LP reserves. The curve receives input `3,994`.

The dynamic fee uses the gross-input selloff window, but is charged in output units and reserved entirely for the protocol. It does not use `protocol_fee_rate` to split its proceeds. Its threshold isolation, piecewise-linear integral, four output segments and rounding are specified in [Dynamic Fee](../for-traders/dynamic-fee.md).

Protocol-fee collection transfers the reserved amounts out of the vault and resets their counters. It does not decrease LP `actual_balance` or `virtual_balance`: those balances already excluded the protocol claim.

### Selloff and surge are part of executable swap math

The curve has no clock, but a complete quote does. The [limiter](../for-traders/max-selloff.md) first projects the input token's buckets using the transaction timestamp, resolves its snapshot-based cap, and accepts **gross** input against that cap. It returns `effective_before`, `effective_after` and `cap`. Base-fee subtraction does not reduce this window usage.

The [surge calculation](../for-traders/dynamic-fee.md#how-the-charged-amount-is-calculated) then uses those three values to locate the taxed span. Its cumulative curve evaluations use `floor(curve_input × consumed_gross_input / gross_input)`. Four equal spans of taxed raw input each have an integrated average rate and an output-token ceiling charge. The sum is subtracted from gross curve output. The rate integral and four output segments are distinct operations; a single averaged rate times total output is not the implementation.

The exact [integer primitive and averaging rules](../for-traders/dynamic-fee.md#integral-and-integer-precision) preserve every fill/normalization floor and rate/amount ceiling. The optional kink changes the primitive inside a segment; it does not change the number of output segments or force their boundaries onto the kink. Curve integration does not establish exact split-trade invariance once nonlinear output allocation, four segments, reserve changes and integer rounding are included.

Use `CubicPoolClient.quoteSwap` for the combined path, or compose the public helpers as in the [executable local example](../for-traders/dynamic-fee.md#reproduce-the-partial-example-with-the-sdk). Neither `calcOutGivenIn` nor `calcSurgeFeePct` alone produces a valid net swap quote.

## Spot prices: specify the direction

Ignoring fees and finite-trade price impact, the marginal **raw output per raw input** is:

```text
spot_raw_out_per_in = V_out × w_in / (V_in × w_out)
```

In human token units:

```text
spot_out_per_in = V_out × w_in / (V_in × w_out) × 10^(decimals_in − decimals_out)
spot_in_per_out = V_in × w_out / (V_out × w_in) × 10^(decimals_out − decimals_in)
```

These are reciprocals. The SDK's `calcSpotPrice` returns **input per output**, scaled by `10^18`. Its `calcSpotOut` returns a raw output amount for a supplied raw input amount. Do not label `calcSpotPrice` as output per input.

For example, virtual reserves of 2 human input tokens and 10 human output tokens with equal weights imply 5 output per input, or 0.2 input per output. With input weight 80% and output weight 20%, those become 20 output per input and 0.05 input per output. Ignoring weights is valid only when those weights are equal.

`quoteSwap.spotOut` uses input **after the base fee**. Its `priceImpactHbps` compares this spot output with net output **after surge**, so this SDK field includes the effect of surge as well as curve price impact. It is not a pure fee-free curve-impact metric:

```text
spot_out        = floor(x × V_out × w_in / (V_in × w_out))
curve_impact    = floor((spot_out − Y) × 1,000,000 / spot_out)
SDK_net_impact  = floor((spot_out − (Y − S)) × 1,000,000 / spot_out)
```

These impact expressions assume positive spot output and a nonnegative loss; the SDK returns zero for nonpositive spot output or output at least equal to spot. Impact scale is `1,000,000 = 100%`, unlike the `10,000` scale used for surge rate points.

In the [partial-window worked example](../for-traders/dynamic-fee.md#worked-example-a-partial-trade-crossing-the-kink), `x = spot_out = 19,940,000`, `Y = 19,550,169`, and `S = 1,210,098` raw output units. Curve-only impact is `19,550 = 1.9550%`; the SDK field is `80,237 = 8.0237%`. The separate floors mean subtracting rounded displayed percentages is not an exact fee reconciliation.

## Seed deposit and initial BPT

The first `add_liquidity`, identified by zero BPT mint supply, is pool-admin-only. It takes the requested basket verbatim and requires at least one positive actual deposit. Slots with zero initial deposit may remain sidelined. The initial BPT amount is derived from configured **virtual** balances, not from the deposit's monetary value; only the defensive all-virtual-zero path uses the deposit basket instead.

The implemented invariant calculation is:

1. Normalize each raw balance to six decimal places: divide by `10^(decimals − 6)` when decimals are at least six, or multiply by `10^(6 − decimals)` otherwise.
2. If any normalized balance is zero, the returned invariant is zero.
3. Start `invariant = ONE`. For every token slot, compute `pow_fp(normalized_balance, weight_fp)`, then `invariant = floor(invariant × powered / ONE)` with a wide intermediate.
4. Convert the returned integer directly to raw `u64` BPT and require it to be at least `1,000` raw BPT and the caller's `minimum_bpt_amount`.

The exact normalization and fixed-point conventions determine BPT supply. Do not substitute a dollar-valued geometric mean or multiply the result by another decimal factor. `calculateInvariant` exposes this calculation; `quoteSeedDeposit` also checks the caller and seed-deposit rules. BPT has 9 decimals, so the supply floor of 1,000 is `0.000001 BPT`.

## Subsequent proportional deposits

Let `a_i` be each pre-deposit LP actual balance, `m_i` the user's maximum offered amounts, and `B` the current BPT supply. A live slot (`a_i > 0`) requires `m_i > 0`; a sidelined slot requires `m_i = 0`.

```text
r_i          = floor(m_i × ONE / a_i)         # Live slots only.
r            = min(r_i)
bpt_minted   = floor(B × r / ONE)
transferred_i= floor(a_i × r / ONE)
virtual_new_i= virtual_i + floor(virtual_i × r / ONE)
```

The contract requires positive minted BPT and the caller's BPT minimum. It transfers only the cropped basket, leaving unused offers in the user's wallet. Sidelined slots are excluded from the ratio but their virtual balances still scale. Window snapshots and both selloff counters also scale by this ratio for enabled limiters.

The two floor operations are separate. A very small offered amount can produce a positive BPT amount while some actual token transfers round to zero. SDK `0.11.1` rejects these zero-transfer live-leg quotes/builds; that client-side check does not change the deployed contract. See [Rounding limit in this contract version](../for-lps/liquidity.md#rounding-limit-in-this-contract-version) for the limitation and who is exposed. Do not infer a universal no-dilution guarantee from the formulas alone.

## Proportional withdrawals

The pool preserves at least 1,000 raw BPT supply. For request `b` and current supply `B`:

```text
effective_burn = min(b, B − 1,000)
r              = floor(effective_burn × ONE / B)
output_i       = floor(actual_i × r / ONE)
virtual_new_i  = virtual_i − floor(virtual_i × r / ONE)
```

A zero request, a request above total supply, or no burnable supply is rejected. The user must own enough BPT for the effective burn. Only that effective amount is burned; excess requested BPT remains in the user's account. Each payout must meet its corresponding minimum.

Do not collapse the ratio and payout into one division `floor(actual_i × effective_burn / B)`: the deployed code performs two floor operations and can return a smaller amount. Protocol-fee counters are unchanged. The virtual-balance and selloff-window proportional reductions include sidelined slots; actual payouts from those slots are zero.

## Single-token deposit allocations

The single-token helper performs real internal swaps followed by a proportional deposit. Its allocation calculation uses the pre-swap pool state:

```text
effective_actual_i = min(actual_i, virtual_i)
scaled_i           = floor(effective_actual_i × ONE / virtual_i)
W_i                = floor(scaled_i × weight_bps_i / 10,000)
allocation_i       = floor(amount_in × W_i / sum(W))
```

Any integer remainder is added to the input slot so allocations sum exactly to the input. These allocations are all **input-token amounts**: a non-input slot's allocation is how much input to swap toward that slot, not how many output tokens will be deposited. The `min(actual, virtual)` clamp prevents an actual reserve larger than its virtual reserve from demanding a proportion of the swap that the curve cannot deliver.

The helper keeps the input slot's allocation and swaps the others in ascending token-index order. Each swap uses the latest reserves and advances the same input token's selloff window. Base and dynamic fees are applied per swap, not once to the total deposit. Sidelined output slots receive no allocation; the input token itself must have actual liquidity.

After swaps, the helper reads actual token receipts and the **post-swap** pool state. It chooses:

```text
r_helper = min(floor(helper_balance_i × ONE / actual_after_swaps_i))
capped_i = floor(actual_after_swaps_i × r_helper / ONE)
```

The proportional pool deposit then recomputes its ratio from this capped basket and applies its own floors. Both stages must be reproduced in a quote. All leftover helper token balances are refunded to the user, and only BPT newly minted by this operation is transferred. A positive final `minimum_bpt_amount` protects the entire sequence; internal swaps use `minimum_amount_out = 0`.

`quoteSingleTokenDeposit` reproduces the sequence on a private state copy. Its optional existing `helperBalances` must be provided when the helper already contains tokens; the default assumes empty helper token accounts. The exported `computeTwoTokenOptimalAllocations` is a deprecated analytical base-fee-only optimizer. It does not represent the deployed helper and does not model dynamic fees or selloff limits.

## Quote boundaries

Quotes use synchronized state and a chosen timestamp; they are not a reservation of liquidity or window capacity. Another transaction or policy update may change the result before execution. The SDK quote path also guards unsupported token extensions and dust deposits more strictly than the mathematical primitives alone. A successful pure-math call is not proof that an instruction's accounts, token programs or authorities are valid.

For submitted swaps, use net `minAmountOut`; for deposits use the final BPT minimum; for withdrawals use every token minimum. Slippage values in the SDK use denominator `1,000,000`, so `5,000 = 0.5%`:

```text
minimum = floor(expected × (1,000,000 − slippage_hbps) / 1,000,000)
```

## Sources

- [Contract swap and proportional liquidity math](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/math/cubic_math.rs)
- [Contract invariant](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/math/weighted_math.rs)
- [Contract fixed-point power](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/math/log_exp_math.rs)
- [Contract single-token allocation and cap](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/single-token-liquidity/src/math.rs)
- [SDK math modules](https://github.com/coffer-so/sdk/tree/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/math)
- [SDK stateful quote math](https://github.com/coffer-so/sdk/blob/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/clients/quote-math.ts)
