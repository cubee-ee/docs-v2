# Dynamic Fee (Surge Fee)

Coffer's dynamic fee increases the cost of selling a token as that token's [max-selloff window](max-selloff.md) fills. It is an **additional fee in the output token**, on top of the pool's input-token base swap fee. The entire dynamic fee is reserved for the protocol. It is not LP fee income.

The implementation documented here is contracts `audit-fixes-excluded-SF` at `96a2ee2` and SDK `0.11.1` at `27de819`. The implemented rate curve is piecewise linear with an optional kink. The unused `SURGE_FEE_CONVEXITY_FP` constant in the contract is not part of this calculation.

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

The rate helper uses the difference of primitives divided by the taxed interval width and **rounds the result up** to the percentage scale. Its fixed-point ramp calculation divides `t²` by `2 × width` before multiplying by the rate span; preserving this operation order matters at integer boundaries. If both rounded fill endpoints are equal above the threshold, it uses the endpoint rate, rounded up. A threshold or full-fill kink would degenerate a denominator, so the validated configuration excludes those positions.

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

`minimum_amount_out` is checked against **net user output after surge**. A quote or route must use that net value when setting slippage protection. A minimum based on gross output may make an otherwise valid trade fail.

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
- [Policy validation](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_max_selloff.rs)
- [SDK surge math](https://github.com/coffer-so/sdk/blob/27de819c469056bfb7cd3ab3a4cfdbde741db2f8/src/math/surgeFee.ts)
