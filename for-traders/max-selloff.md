# Max-Selloff Window

A pool can limit how much of each token is **sold into the pool**. The limit is a percentage of a stored virtual-balance snapshot, enforced with two time buckets. A swap that exceeds it fails with `MaxSelloffExceeded`. This is separate from the [dynamic fee](dynamic-fee.md), which can increase the cost of an allowed swap as the same window fills.

This page describes contracts `audit-fixes-excluded-SF` at `96a2ee2` and SDK `0.11.1` at `09cc776`.

## What is limited

The input token's policy applies to the swap's **gross `amount_in`**, before the base swap fee. Buying that token as the output does not add to its selloff counters. The retained input-token portion of a single-token deposit does not count as a sale; each internal swap does, in execution order.

`max_selloff_pct` is an integer percentage with scale **10,000 = 100%**. For example, `1,000` means a cap of 10% of the snapshot. It is not an amount in raw token units. The resolved cap and counters are raw units of the input token:

```text
cap = floor(max_selloff_pct × selloff_vb_snapshot / 10,000)
```

A virtual balance is a pricing reserve; it need not equal the actual token balance in the vault. The cap is consequently not a percentage of vault holdings, circulating supply, or dollar value. Output liquidity and price impact remain independent constraints.

- `max_selloff_pct = 0` disables both the window check and dynamic fee for that token. Its counters are not advanced by swaps while disabled.
- `max_selloff_pct > 0` requires a positive period. The percentage cannot exceed `10,000`.
- A small percentage and snapshot can resolve to a zero cap after flooring. Every positive sale then exceeds it.
- Equality is accepted: the post-swap effective amount may equal the cap.

## Stored state

Each token's `dynamics` row contains:

| Contract field | SDK `PoolTokenInfo` field | Meaning |
|---|---|---|
| `previous_selloff` | `previousSelloff` | Prior bucket's amount, possibly rescaled when its balance basis changes |
| `current_selloff` | `currentSelloff` | Current bucket's accumulated gross inputs, possibly rescaled by LP actions |
| `window_start_timestamp` | `windowStartTimestamp` | Start of the current bucket, Unix seconds |
| `selloff_vb_snapshot` | `selloffVbSnapshot` | Virtual-balance basis used to resolve this window's cap |

The SDK exposes these four values as `BN`. The policy fields `maxSelloffPct` and `maxSelloffPeriodLength` are numbers. Read complete state with `CubicPoolClient.sync()` before quoting; do not infer the state only from a volume chart.

### An unopened window is not a disabled window

A stored `selloff_vb_snapshot = 0` means the balance basis has not yet been initialized. With a positive `maxSelloffPct`, the next check resolves it from the **current pre-swap virtual balance**. A new or newly migrated token can therefore already have a usable cap and dynamic-fee curve even though no sell has initialized its snapshot. A UI should project the next check instead of hiding the limit because the stored snapshot is zero.

For example, `maxSelloffPct = 1,000`, live virtual balance 1,000 tokens and zero snapshot resolve to a 100-token cap. By contrast, a live balance of one raw unit resolves to `floor(1 × 1,000 / 10,000) = 0`: the policy is enabled, but no positive sale fits. The enable/disable switch is the configured percentage, not whether the resolved cap is nonzero.

## Rotation and acceptance

The contract reads `Clock::unix_timestamp`. Compute `elapsed = max(0, now − window_start_timestamp)`; a backward clock movement never rotates a window backward.

| Elapsed time | Candidate bucket update | Snapshot |
|---|---|---|
| Less than one period | Keep both buckets and their start | Keep the stored snapshot, unless zero, in which case capture the current pre-swap virtual balance |
| At least one but less than two periods | Move `current` into `previous`, set `current = 0`, advance start by exactly one period | Capture the current pre-swap virtual balance |
| At least two periods | Clear both buckets and set start to `now` | Capture the current pre-swap virtual balance |

When a rotation changes an initialized snapshot, the carryover is converted to the new basis:

```text
previous = floor(candidate_previous × new_snapshot / old_snapshot)
```

After the candidate rotation, let `e` be elapsed time within the new current bucket:

```text
weighted_previous = floor(previous × (period − e) / period)
effective_before  = weighted_previous + current
effective_after   = effective_before + gross_amount_in

require effective_after <= cap
```

Only on acceptance are the new buckets, timestamp and snapshot stored, with `current += gross_amount_in`. A rejected check commits none of its candidate changes. If a later fee, liquidity, slippage or token-transfer check fails, Solana transaction atomicity also rolls back the window update.

For a fixed initialized current bucket and unchanged policy/liquidity, the theoretical headroom at elapsed time `e` is:

```text
headroom(e) = cap − current − floor(previous × (period − e) / period)
```

A negative result means even a zero-input helper check fails; there is no positive headroom. Do not clamp a negative result and then treat it as an unrestricted policy. At the next boundary the cap can change because the live virtual balance is captured, so extrapolating this formula past that boundary without rebasing is incorrect.

This is a **two-bucket approximation**, not an exact record of every sale in the preceding `period` seconds. The previous bucket fades linearly; the current bucket does not decay until it becomes the previous one. One full period without a new sale therefore does not guarantee an entirely empty limiter. Two periods from the stored bucket start make both buckets stale, absent further successful updates.

## Why the snapshot changes

Within a window, ordinary swaps do not continuously enlarge the cap as the input virtual balance grows. The stored basis stays fixed until rotation. Buying the token can shrink its live virtual balance without changing the snapshot until the next rotation.

Proportional **add/remove liquidity** are handled differently. The contract scales the snapshot and both counters together with the liquidity ratio, keeping the fraction of capacity used approximately unchanged, subject to integer rounding:

```text
add:    value_new = value + floor(value × ratio_fp / 10^18)
remove: value_new = value − floor(value × ratio_fp / 10^18)
```

Here `value` is each of `selloff_vb_snapshot`, `previous_selloff` and `current_selloff`; the timestamp is unchanged. This applies only to tokens whose cap is enabled. Scaling both capacity and usage prevents adding then withdrawing liquidity from leaving an artificially large cap behind. A seed deposit does not run this proportional rescaling path.

The `ratio_fp` is the actual liquidity instruction's fixed-point ratio: proportional joins take the minimum offered/live-actual ratio, and withdrawals use the effective burned-BPT ratio. It is not a dollar-value ratio and not a percentage inferred from the vault balance. The [liquidity math](../technical/math.md#subsequent-proportional-deposits) has the exact floors.

Administrative virtual-balance changes do not call this LP rescaling helper. The new live basis is captured on a later window rotation, with the carryover conversion above. Policy changes also preserve the existing window state.

## Worked example: crossing a boundary

Token X has 6 decimals. Initially:

```text
max_selloff_pct = 1,000          # 10%
period = 60 seconds
snapshot = 1,000,000,000 raw    # 1,000 X
cap = 100,000,000 raw           # 100 X
previous = 0; current = 80,000,000; start = 0
```

At `t = 30`, before any rotation, another 20 X is allowed and 20.000001 X is not. The current bucket's 80 X has not decayed just because half its period has passed.

For a separate boundary example, keep the original 80 X current bucket. At `t = 90`, assume the live virtual balance is now 1,200 X:

1. Rotate once: `start = 60`, candidate previous = 80 X, current = 0.
2. Capture 1,200 X and rebase previous: `80 × 1,200 / 1,000 = 96 X`.
3. Cap becomes 120 X. Half the new period has elapsed, so weighted previous is `96 × 30 / 60 = 48 X`.
4. Headroom is `120 − 48 = 72 X`. A sale of 72 X reaches the cap exactly; a larger sale fails.

For an LP rescaling example, a 50% proportional liquidity removal from an initialized window with snapshot 1,000 X and current usage 80 X produces snapshot 500 X and current usage 40 X. The cap falls from 100 X to 50 X; the used fraction remains 80%, before rounding effects.

## Quoting in the SDK

`quoteSwap` runs the limiter, dynamic fee and AMM math together. It uses the chain timestamp fetched by `sync()` unless an explicit `nowSeconds` argument is supplied. It does not advance the public cache when quoting. Another trade, a policy change or the eventual execution timestamp can change the result.

For a separate headroom calculation, the exported helper performs the same rotation and rebasing. This example assumes an already synchronized `pool` of type `PoolInfo`:

```typescript
import { checkAndAdvanceSelloff } from "@cubee_ee/sdk";

const token = pool.tokens[tokenInIndex];
if (pool.chainTimestamp === undefined) throw new Error("Sync the pool first");
if (token.maxSelloffPct > 0 && (
  token.maxSelloffPeriodLength === undefined || !token.previousSelloff ||
  !token.currentSelloff || !token.windowStartTimestamp || !token.selloffVbSnapshot
)) throw new Error("Missing selloff state");

const observed = checkAndAdvanceSelloff({
  state: {
    previousSelloff: BigInt(token.previousSelloff?.toString() ?? "0"),
    currentSelloff: BigInt(token.currentSelloff?.toString() ?? "0"),
    windowStartTimestamp: BigInt(token.windowStartTimestamp?.toString() ?? "0"),
    selloffVbSnapshot: BigInt(token.selloffVbSnapshot?.toString() ?? "0"),
  },
  maxSelloffPct: token.maxSelloffPct,
  period: token.maxSelloffPeriodLength ?? 0,
  amountIn: 0n, // Helper calculation only; an actual zero-input swap is invalid.
  virtualBalance: BigInt(token.virtualBalance.toString()),
  now: BigInt(pool.chainTimestamp),
});

const headroom = observed === null
  ? null // Disabled; this does not mean unlimited output liquidity.
  : observed.maxSelloffCap - observed.effectiveSelloffBefore;
```

`observed.vbSnapshot` and `observed.maxSelloffCap` are the **candidate** basis and cap at the supplied timestamp. `observed.effectiveSelloffBefore` is usage before adding this hypothetical input; `observed.effectiveSelloff` includes it. For the headroom call above they are equal because `amountIn` is zero. A displayed fill should divide by this resolved cap and separately represent a zero-cap enabled policy; dividing counters by a zero stored snapshot is not a valid display calculation.

The helper throws `MaxSelloffExceeded` if the already-used amount exceeds a newly tightened cap, even for this zero-input calculation. No positive sale fits at that timestamp. Do not replace the check with `percentage × live_virtual_balance`, or assume missing counters are zero for an enabled policy.

## Configuration and events

The pool admin calls `set_max_selloff(params: Vec<SelloffParams>)` with one complete entry for every token slot in pool order, including sidelined slots. The old two-vector raw-amount API is obsolete. See [Dynamic Fee: policy fields](dynamic-fee.md#policy-fields-and-units) for all seven fields, bounds and the SDK builder.

Changing the percentage or period **does not reset counters or the snapshot**. A smaller cap can block further sells immediately. A new period is applied to the existing timestamp on the next check. Disabling and later re-enabling also does not itself clear old state; rotation determines whether it is stale. The setter is pool-admin-only, and a renounced pool admin cannot use it.

A successful swap with an enabled window emits `MaxSelloffWindowAdvanced`: pool, input token index, effective amount including the swap, resolved cap, snapshot, both buckets, bucket start and transaction timestamp. It is absent when the limiter is disabled. `MaxSelloffSet` describes a policy update, but its current event layout omits the mid-rate and kink fields; read the pool account for the complete policy. LP rescaling does not emit `MaxSelloffWindowAdvanced`.

## Sources

- [Contract limiter and LP rescaling](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/math/max_selloff.rs)
- [Policy instruction and validation](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_max_selloff.rs)
- [SDK limiter](https://github.com/coffer-so/sdk/blob/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/math/maxSelloff.ts)
