# Dynamic Fee (Surge Fee)

Coffer's dynamic fee increases the cost of selling a token as that token's [max-selloff window](max-selloff.md) fills. It is an **additional fee in the output token**, on top of the pool's input-token base swap fee. The entire dynamic fee is reserved for the protocol. It is not LP fee income.

The implementation documented here is contracts `audit-fixes-excluded-SF` at `96a2ee2` and SDK `0.11.1` at `09cc776`. The implemented rate curve is piecewise linear with an optional kink. The unused `SURGE_FEE_CONVEXITY_FP` constant in the contract is not part of this calculation.

## What triggers the fee

Each input token has its own policy. The dynamic fee uses the same effective gross-input usage and snapshot-based cap as the limiter:

```text
fill = effective_selloff / resolved_cap
```

The fee is zero if the limiter is disabled (`max_selloff_pct = 0`), the high rate is zero, or the trade's traversed fill remains at or below the threshold. A configured positive high rate requires a threshold below 100%. A trade that would exceed the hard cap still fails: paying the dynamic fee does not buy additional capacity.

The fee depends on the **input token's** window and policy. Its amount is denominated in the **output token**. The output token's own selloff policy is not charged for buying it.

## Policy fields and units

The complete `SelloffParams` entry is supplied to `set_max_selloff`. All entries must be present in pool order; the vector length equals `token_count`.

| Instruction field | SDK field | Meaning and validation |
|---|---|---|
| `max_selloff_pct: u16` | `maxSelloffPct` | Fraction of the virtual-balance snapshot used as the hard cap; `0…10,000`, where `1,000 = 10%`; zero disables limiter and surge |
| `period_length: u32` | `periodLength` | Seconds; positive if the cap is enabled; otherwise zero is permitted |
| `fee_threshold_pct: u16` | `feeThresholdPct` | Fill where charging begins; `8,000 = 80%` of the cap; must be `< 10,000` when high rate is positive |
| `fee_slope_low_pct: u16` | `feeSlopeLowPct` | Fee rate immediately above the threshold; `200 = 2%` |
| `fee_slope_high_pct: u16` | `feeSlopeHighPct` | Fee rate at full window fill; `3,000 = 30%`; zero disables surge |
| `fee_slope_mid_pct: u16` | `feeSlopeMidPct` | Fee rate at the kink; `1,000 = 10%` |
| `fee_kink_pct: u8` | `feeKinkPct` | Kink position in **whole percent** of window fill: `90 = 90%`; zero selects a single straight line |

The three rate values must satisfy `0 <= low <= mid <= high <= 10,000`. Despite their historical `slope` names, these are **rates at curve points**, not derivatives. `mid` is ignored in the single-line calculation, but its ordering is still validated.

A nonzero kink must be strictly inside the taxed span: `kink < 100` and `100 × kink > threshold`. A value of 100 is rejected by the setter. A kink of zero means no kink, not a kink at 0%.

These validations apply even when the limiter itself is disabled. If `high = 0`, the ordering forces `low = mid = 0`; the threshold may retain any `u16` value, subject to the kink constraint. Such a threshold has no charging effect. Prefer an ordinary `0…10,000` threshold in disabled policy displays rather than interpreting an arbitrary stored value as a live rate.

Example policy for one token:

```typescript
import { buildSetMaxSelloffIx, type SelloffParams } from "@cubee_ee/sdk";

const policy: SelloffParams = {
  maxSelloffPct: 1_000,    // Cap = 10% of snapshot.
  periodLength: 3_600,     // One hour.
  feeThresholdPct: 8_000, // Surge starts after 80% of cap is used.
  feeSlopeLowPct: 0,       // 0% just above threshold.
  feeSlopeHighPct: 3_000, // 30% at full cap.
  feeSlopeMidPct: 1_000,  // 10% at kink.
  feeKinkPct: 90,          // Kink at 90% fill, NOT 0.90%.
};

// cfg, poolAddress and poolAdmin are the existing config and public keys.
// policies contains exactly one entry per token slot in pool order.
const instruction = buildSetMaxSelloffIx(cfg, poolAddress, poolAdmin, policies);
```

### Reading the policy from a pool account

The setter's field names differ from the stored `AssetConfig` fields and parsed SDK pool state. Do not look for `feeThresholdPct` on a synchronized token row:

| Setter / SDK `SelloffParams` | Stored `AssetConfig` | SDK `PoolTokenInfo` |
|---|---|---|
| `maxSelloffPct` | `max_selloff_pct` | `maxSelloffPct` |
| `periodLength` | `max_selloff_period_length` | `maxSelloffPeriodLength` |
| `feeThresholdPct` | `variable_fee_threshold_pct` | `variableFeeThresholdPct` |
| `feeSlopeLowPct` | `variable_fee_slope_low_pct` | `variableFeeSlopeLowPct` |
| `feeSlopeMidPct` | `variable_fee_slope_mid_pct` | `variableFeeSlopeMidPct` |
| `feeSlopeHighPct` | `variable_fee_slope_high_pct` | `variableFeeSlopeHighPct` |
| `feeKinkPct` | `variable_fee_kink_pct` | `variableFeeKinkPct` |

The stored policy describes the curve; the four window fields in [Stored state](max-selloff.md#stored-state) determine the position on that curve. A threshold of 80% is **80% of the cap**, not 80% of the virtual balance: a 10% cap and an 80% threshold start charging around 8% of the snapshot, subject to integer rounding.

The builder creates an instruction; it does not sign or submit it. The pool admin must authorize the transaction. The setter preserves the window counters, timestamp and snapshot. See [Max-Selloff Window](max-selloff.md#configuration-and-events) for the immediate consequences of policy changes.

## Rate curve

Let `f` be window fill as a real fraction from 0 to 1, `T` the threshold, and `L`, `M`, `H` the three fee rates as fractions.

Without a kink:

```text
f <= T:   rate(f) = 0
f > T:    rate(f) = L + (H − L) × (f − T) / (1 − T)
```

With a valid kink `K`:

```text
f <= T:        rate(f) = 0
T < f <= K:    rate(f) = L + (M − L) × (f − T) / (K − T)
K < f <= 1:    rate(f) = M + (H − M) × (f − K) / (1 − K)
```

At the threshold itself the charged rate is zero. If `L > 0`, the rate jumps to the low-rate level immediately above it. The curve is non-decreasing, but validation does **not** require the second line to be steeper than the first; rates and segment widths determine that.

For the example policy, the marginal rates are:

| Window fill | Rate |
|---|---|
| Up to and including 80% | 0% |
| 85% | 5% |
| 90% | 10% |
| 95% | 20% |
| 100% | 30% |

These are marginal rates at a position, not the percentage charged on the entire swap output. A swap traverses an interval, which is integrated as described next.

## How the charged amount is calculated

A trade can cross the threshold and kink in one instruction. Charging its final rate on the entire output would tax earlier portions at a rate they never traversed. The contract instead charges only the output produced above the threshold, in **four segments**.

Let:

- `B` = effective usage before the trade, in raw input units.
- `A` = effective usage after adding gross input; `A − B = amount_in`.
- `C` = resolved raw-input cap.
- `x` = input after subtracting the base swap fee.
- `Y` = gross AMM output before subtracting surge.
- `Q(x)` = the AMM's cumulative output for fee-adjusted input `x`, using the pre-swap balances and weights.

The algorithm is:

```text
threshold_units = floor(C × fee_threshold_pct / 10,000)
taxed_start     = max(B, threshold_units)

u[0] = taxed_start
u[k] = taxed_start + floor((A − taxed_start) × k / 4), for k = 1, 2, 3
u[4] = A

cumulative_output(u) =
  0                                      when u <= B
  Y                                      when u >= A
  Q(floor(x × (u − B) / (A − B)))         otherwise

segment_output[k] = max(0, cumulative_output(u[k]) − cumulative_output(u[k−1]))
segment_fee[k]    = ceil(segment_output[k] × average_rate[k] / 10,000)

surge_fee_amount = min(Y, sum(segment_fee))
user_amount_out  = Y − surge_fee_amount
```

The four boundaries divide the **taxed raw-input span equally**. They are not four fixed global fill bands, and they are not automatically aligned to the kink. A segment that straddles the kink uses both lines of the integral. Every cumulative `Q` evaluation uses the same pre-swap balances; these are slices of one swap, not four separately executed swaps.

Zero-width or zero-output segments add no fee. The whole-window rate helper first short-circuits a trade whose rounded fill range is entirely untaxed. The average rate for each remaining segment is the integral over its **taxed fill range**, not the endpoint rate. Input between `B` and the threshold produces untaxed output; its share is found by evaluating the curve, not by multiplying total output by a volume percentage.

### Integral and integer precision

Internally `fill_units = min(floor(effective × 10,000 / C), 10,000)`. Fill therefore has 0.01 percentage point resolution. The taxed span is normalized to `t = (fill − threshold) / (10,000 − threshold)`, stored with `10^18` precision.

For a single line, the primitive is:

```text
F(t) = L × t + (H − L) × t² / 2
```

With a normalized kink `k`:

```text
t <= k: F(t) = L × t + (M − L) × t² / (2k)
t > k:  F(t) = k × (L + M) / 2
                 + M × (t − k)
                 + (H − M) × (t − k)² / (2(1 − k))
```

Those expressions explain the shape. To reproduce the implementation, retain the following integer operations. Here `P = 10,000`, `ONE = 10^18`, `T`, `L`, `M`, `H` are the stored percentage-scale integers, and `kink` is the stored whole-percent integer. All arguments are nonnegative; `ceildiv(n, d) = floor(n / d) + (n mod d != 0 ? 1 : 0)`.

```text
d = P − T
k = 0                                  # Single line when kink = 0.
k = floor((100 × kink − T) × ONE / d)   # Otherwise, for a valid kink.

ramp(z, span, width) = span × floor(z² / (2 × width))
                      # Zero when z, span or width is zero.

F(t), with t clamped to ONE:
  k = 0:   L × t + ramp(t, H − L, ONE)
  t <= k:  L × t + ramp(t, M − L, k)
  t > k:   floor(k × (L + M) / 2)
             + M × (t − k) + ramp(t − k, H − M, ONE − k)

f0 = min(floor(segment_before × P / C), P)
f1 = min(floor(segment_after  × P / C), P)
if f1 <= T: average_rate = 0
otherwise:
  t1 = floor((f1 − T) × ONE / d)
  if f1 > f0:
    f0c = max(f0, T)
    t0  = floor((f0c − T) × ONE / d)
    integral = max(F(t1) − F(t0), 0)
    average_rate = min(ceildiv(d × integral, (f1 − f0c) × ONE), P)
  otherwise:
    average_rate = integer_endpoint_rate(t1)
```

For the endpoint fallback, the integer rate is `L + ceildiv((H − L) × t, ONE)` for one line; `L + ceildiv((M − L) × t, k)` before the kink; or `M + ceildiv((H − M) × (t − k), ONE − k)` after it. Clamp the result to `P`. The fallback is used when a positive raw-input segment does not advance the rounded fill; it prevents that segment's rate from disappearing merely because the window is large.

The fixed-point ramp divides **before** multiplying by the rate span. Moving the division to the end can change the primitive and fee. The primitive, normalized positions and fill have floors; the final average rate and each output-token segment charge have separate ceilings. A raw input amount just above the theoretical threshold can still have `f1 <= T` after fill rounding and therefore pay zero surge. This is why a floating-point graph is explanatory, not an executable quote.

Before these steps, cap zero, high zero or `T >= P` returns zero from the rate helper. This does not make a positive sale valid under an enabled zero-cap limiter: the limiter rejects it first. For malformed legacy curve state, the low-level math treats a kink outside the taxed span as no kink and clamps negative rate spans to zero; the setter rejects those configurations for new policy updates.

The piecewise-linear integral avoids numerical quadrature of the rate curve. Converting that curve into an **output-denominated charge** is still a four-segment approximation because the AMM output is nonlinear in input. Integer rounding and segmentation mean split trades need not pay exactly the same total surge fee as one large trade. Do not advertise exact path independence or a universal percentage bound. Extremely asymmetric weights can also deliver most output before the threshold, leaving little output on which to charge surge; the hard selloff cap remains a separate control.

## Worked example with exact raw amounts

Consider two 6-decimal tokens, equal 50/50 weights, virtual and LP actual balances of 1,000 tokens each. The initialized input snapshot is also 1,000 tokens. Use the example policy above, start with empty buckets, and sell 100 input tokens within the current window.

```text
gross input          = 100,000,000 raw
cap                  = 100,000,000 raw      # 10% of 1,000 tokens
base rate            = 3,000 / 1,000,000    # 0.3%
base fee             = 300,000 raw input   # 0.3 input token
protocol share       = 2,000 / 10,000      # 20% of base fee
protocol input fee   = 60,000 raw input    # 0.06 input token
curve input          = 99,700,000 raw
```

The trade moves from 0% to 100% of the cap. Output produced up to 80% fill is 73,868,267 raw; it is untaxed. The four taxed segments are:

| Fill interval | Segment output, raw | Average rate | Ceiling fee, raw output |
|---|---:|---:|---:|
| 80% → 85% | 4,256,084 | 250 = 2.5% | 106,403 |
| 85% → 90% | 4,217,146 | 750 = 7.5% | 316,286 |
| 90% → 95% | 4,178,738 | 1,500 = 15% | 626,811 |
| 95% → 100% | 4,140,854 | 2,500 = 25% | 1,035,214 |

```text
gross curve output = 90,661,089 raw = 90.661089 output tokens
surge fee          =  2,084,714 raw =  2.084714 output tokens
user receives      = 88,576,375 raw = 88.576375 output tokens
```

These values were checked with the SDK implementation and, for this equal-weight example, independently with the rational constant-product formula `floor(vb_out × x / (vb_in + x))`. Use the SDK fixed-point implementation for general weights rather than assuming that shortcut matches all rounding cases.

The protocol accrues **0.06 input token plus 2.084714 output tokens**. LPs retain 0.24 input token from the base fee. Token units differ; these amounts cannot be added into a meaningful monetary total without prices.

## Worked example: a partial trade crossing the kink

Use the same balances, weights, cap and fee policy, but now assume the effective pre-trade usage is **75 input tokens** and sell **20 input tokens**. This is a separate pre-swap state example, not a second trade following the previous example.

```text
window usage          = 75,000,000 → 95,000,000 raw = 75% → 95% fill
base fee              = 60,000 raw input
protocol input share  = 12,000 raw input
curve input           = 19,940,000 raw input
untaxed curve output  = 4,960,273 raw output       # Produced up to 80% fill.
```

The taxed span is 80%→95%. Dividing it into four equal input spans gives **3.75 percentage points each**, so the third segment crosses the 90% kink:

| Fill interval | Segment output, raw | Average rate after ceiling | Ceiling fee, raw output |
|---|---:|---:|---:|
| 80% → 83.75% | 3,688,031 | 188 = 1.88% | 69,335 |
| 83.75% → 87.5% | 3,660,793 | 563 = 5.63% | 206,103 |
| 87.5% → 91.25% | 3,633,857 | 959 = 9.59% | 348,487 |
| 91.25% → 95% | 3,607,215 | 1,625 = 16.25% | 586,173 |

For the crossing segment, the fee rises from 7.5% to 10% over 2.5 fill percentage points, then from 10% to 12.5% over 1.25 points. Its continuous average is `(2.5 × 8.75% + 1.25 × 11.25%) / 3.75 = 9.583333…%`; the integer rate becomes **9.59%** before applying the output-token ceiling. The first two segment averages also round up from 1.875% and 5.625%.

```text
gross curve output = 19,550,169 raw = 19.550169 output tokens
surge fee          =  1,210,098 raw =  1.210098 output tokens
user receives      = 18,340,071 raw = 18.340071 output tokens
minimum at 0.5%    = 18,248,370 raw = 18.248370 output tokens
```

The endpoint marginal rate is 20%, but that is not the trade's flat fee. The actual surge charge is calculated from the four segment outputs above. These amounts were checked independently with rational equal-weight output and piecewise-linear areas, including both ceiling stages.

## Reproduce the partial example with the SDK

This calculation is local and does not send a transaction. It uses the public math exports in SDK `0.11.1`; `bigint` values remain in raw token units throughout:

```typescript
import {
  checkAndAdvanceSelloff, calculateSwapFee, calculateProtocolFee,
  calcOutGivenIn, calcSurgeFeeAmount, calcSpotOut,
  applySlippage, priceImpactHbps,
} from "@cubee_ee/sdk";

const reserves = {
  virtualBalanceIn: 1_000_000_000n,
  virtualBalanceOut: 1_000_000_000n,
  actualBalanceOut: 1_000_000_000n, // Already excludes protocol fees.
  weightInBps: 5_000n,
  weightOutBps: 5_000n,
};
const amountIn = 20_000_000n;
const window = checkAndAdvanceSelloff({
  state: {
    previousSelloff: 0n, currentSelloff: 75_000_000n,
    windowStartTimestamp: 0n, selloffVbSnapshot: 1_000_000_000n,
  },
  maxSelloffPct: 1_000, period: 3_600,
  amountIn, virtualBalance: reserves.virtualBalanceIn, now: 0n,
});
const feeAmount = calculateSwapFee(amountIn, 3_000);
const protocolFeeAmount = calculateProtocolFee(feeAmount, 2_000);
const amountInAfterFee = amountIn - feeAmount;
const grossAmountOut = calcOutGivenIn({ ...reserves, amountIn: amountInAfterFee });
const surgeFeeAmount = calcSurgeFeeAmount({
  ...reserves, window, amountInAfterFee, amountOut: grossAmountOut,
  thresholdPct: 8_000, slopeLowPct: 0,
  slopeMidPct: 1_000, slopeHighPct: 3_000, kinkPct: 90,
});
const amountOut = grossAmountOut - surgeFeeAmount;
const spotOut = calcSpotOut({ ...reserves, amountIn: amountInAfterFee });
const minAmountOut = applySlippage(amountOut, 5_000);
console.log({
  feeAmount, protocolFeeAmount, grossAmountOut, surgeFeeAmount,
  amountOut, minAmountOut, priceImpactHbps: priceImpactHbps(spotOut, amountOut),
});
```

For a real pool, prefer `CubicPoolClient.sync()` followed by `quoteSwap`: it also checks enabled flags, token indices, input activity, complete window state and supported token extensions. Replace the example reserves, bucket state, policy and time with the synchronized values when building a separate calculator. Never run the pure curve on gross input, use the output token's surge policy, or subtract protocol-fee counters from `actualBalanceOut`.

## Accounting and slippage

The pool stores `actual_balance` as LP-owned reserves, excluding protocol fees. For a swap:

```text
input actual and virtual balances += amount_in − protocol_input_fee
input protocol_fees_owed          += protocol_input_fee

output actual and virtual balances -= gross_curve_output
output protocol_fees_owed          += surge_fee_amount
output vault transfer to user      = gross_curve_output − surge_fee_amount
```

The surge fee stays in the vault, reserved for the protocol. Under normal accounted transfers, `vault token amount = actual_balance + protocol_fees_owed`; unsolicited token donations can make the vault hold additional unaccounted tokens. Do not subtract `protocol_fees_owed` again from the stored actual balance.

`minimum_amount_out` is checked against **net user output after surge**. A quote or route must use that net value when setting slippage protection. A minimum based on gross output may make an otherwise valid trade fail. Slippage tolerance is not another fee: at 0.5%, the partial example still expects 18.340071 tokens, while 18.248370 is the smallest acceptable execution output.

The SDK's impact field also uses net output. In that example, the base-fee-adjusted spot output is 19.94 tokens: curve-only impact is `floor((19.94 − 19.550169) / 19.94 × 1,000,000) = 19,550`, or **1.9550%**; SDK `priceImpactHbps` is `80,237`, or **8.0237%**, after including surge. Dividing the surge charge by the whole swap output is neither the endpoint rate nor the SDK price-impact metric.

## SDK fields and event interpretation

`CubicPoolClient.quoteSwap(inputIndex, outputIndex, amountIn, slippageHundredthsBps?, nowSeconds?)` includes:

| Quote field | Units / meaning |
|---|---|
| `feeAmount` | Base fee in raw input units |
| `protocolFeeAmount` | Protocol's share of the base fee, raw input units |
| `grossAmountOut` | Curve output before surge, raw output units |
| `surgeFeeAmount` | Additional protocol fee, raw output units |
| `amountOut` | What the user receives, raw output units |
| `minAmountOut` | Net output after applying the chosen slippage tolerance |

The on-chain `Swap` event's `amount_out` is also net of surge. Its `fee_amount` and `protocol_fee_amount` are input-token amounts; `surge_fee_amount` is an output-token amount. Gross output is `amount_out + surge_fee_amount`.

`calcSurgeFeePct` returns a percentage-scale integer, not a token amount. Multiplying it by the entire gross output is not a replacement for `calcSurgeFeeAmount`, which implements threshold isolation and four output segments. The single-token deposit quote uses this fee calculation for each internal swap, advances a private copy of window and reserve state, and protects final BPT output. See [Single-Token Deposit](../sdk/single-token-deposit.md).

## Sources

- [Rate curve and integral](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/math/surge_fee.rs)
- [Swap charging, transfers and events](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/swap.rs)
- [Stored policy and window layout](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/state/cubic_pool.rs)
- [Policy validation](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_max_selloff.rs)
- [SDK surge math](https://github.com/coffer-so/sdk/blob/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/math/surgeFee.ts)
