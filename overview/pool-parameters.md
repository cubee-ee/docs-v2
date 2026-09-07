# Pool Parameters

This reference describes contracts [`audit-fixes-excluded-SF` at `96a2ee2`](https://github.com/coffer-so/contracts/tree/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs) and SDK [`0.11.1` at `09cc776`](https://github.com/coffer-so/sdk/tree/09cc7766a1e865b5c3f9b97a0526a982671683f5/src). Defaults below are values written by this code; an existing account may contain different settings.

Amounts are raw token integers unless explicitly stated. A token with 6 decimals represents one whole token as `1_000_000`; BPT uses 9 decimals. Use integer arithmetic and preserve `u64` values as `BN` or `bigint` in JavaScript. See [all account fields](../technical/accounts-events.md).

## Creation and identity

| Parameter | On-chain rule | Later changes |
| --- | --- | --- |
| Config | Pool stores its config pubkey; config is part of its PDA seeds | Pool cannot switch configs |
| Pool ID | `u64`, unique together with config; PDA seeds `["cubic_pool", config, pool_id_le]` | Fixed |
| Token set and order | 2–10 unique mints, defined by ordered remaining mint accounts | Fixed; slots do not get removed |
| Token programs | Classic SPL Token or Token-2022, stored per token | Fixed |
| Mint decimals | Must be at most 18 at creation | Read from mint, not stored on the pool |
| BPT | PDA `["bpt_mint", pool]`, pool is mint authority, 9 decimals | Supply changes on joins and exits |
| Initial virtual balances | One positive `u64` per token | Move with swaps/LP operations; configured range manager can change them |
| Weights | One `u64` per token, 100–9,900, sum exactly 10,000 | Range manager can change them within its configured envelope |
| Pool admin | Creation payer | Two-step transfer or permanent renunciation |
| Token extension policy | Creator override or config default, OR effective hard floor | Stored creation-policy snapshot; migration can backfill zero |

Weights and virtual balances are not immutable. “Leverage” is the ratio `virtual_balance / actual_balance`; the contract stores the two balances rather than a standalone target-leverage field. The ratio is undefined for a zero actual reserve. Off-chain UI limits are not contract limits.

A newly initialized pool has zero actual reserves, zero protocol-fee counters,
and zero BPT supply. `pool_enabled`, `swaps_enabled`, and every live token's
`is_active` start true. The pending pool admin, range-manager fields and ALT
address start zero. Selloff policies/buckets/snapshots start zero, while each
slot's `window_start_timestamp` is set to the initialization time. These are
creation defaults, not the state of an existing pool.

## Fees and switches

| Parameter | Type and units | Rule | Who controls it |
| --- | --- | --- | --- |
| `swap_fee_rate` | `u32`, denominator 1,000,000 | 0–100,000; `3_000` = 0.3%, cap 10% | Pool admin |
| `protocol_fee_rate` | `u16`, denominator 10,000 | 0–5,000; share of base swap fee, cap 50% | Config protocol authority |
| `default_protocol_fee_rate` | Config `u16`, same scale | Explicit config-init argument copied to new pools; constant 2,000 is not an automatic override of that argument | Treasury config creation |
| `pool_enabled` | `bool` | False blocks swaps, LP add/remove, single-token deposits and range-manager updates | Protocol authority; supervisor through batch freeze/unfreeze |
| `swaps_enabled` | `bool` | False blocks swaps and single-token deposits; proportional LP add/remove remain available | Pool admin or protocol authority |
| `is_active[i]` | Per-token `bool` | False blocks this token as swap **input**; output and proportional LP operations remain possible | Pool admin or protocol authority; supervisor wrapper |

Both base fee and the protocol's share round upward. The LP receives the base fee less the protocol's share. Surge fee is a separate output-token charge, routed entirely to the protocol; it is not another value of `swap_fee_rate`. See [math](../technical/math.md).

## Selloff and dynamic fee policy

`set_max_selloff(params)` replaces the policy for all `token_count` tokens in pool order. A parameter object is serialized in this exact order:

| Argument field | Type | Units and validation |
| --- | --- | --- |
| `max_selloff_pct` | `u16` | 10,000 = 100% of virtual-balance snapshot; at most 10,000; zero disables limiter and surge |
| `period_length` | `u32` | Seconds; positive when limiter is enabled |
| `fee_threshold_pct` | `u16` | Window-fill scale 10,000 = 100%; below 10,000 when surge is enabled |
| `fee_slope_low_pct` | `u16` | Output fee rate at the threshold, denominator 10,000 |
| `fee_slope_high_pct` | `u16` | Output fee rate at full window fill, at most 10,000; zero disables surge |
| `fee_slope_mid_pct` | `u16` | Output fee rate at kink; `low ≤ mid ≤ high` |
| `fee_kink_pct` | `u8` | **Whole percent**, unlike other `_pct` fields; zero means no kink; otherwise `threshold < 100 × kink < 10,000` |

The stored counterparts are `max_selloff_pct`, `max_selloff_period_length`, and `variable_fee_*`. New pools start with zero limits and zero surge parameters. Enabling surge requires an enabled limiter and nonzero high rate. With no kink, the curve is a straight line between low and high; with a kink it has two straight sections. It is not the old exponential curve.

The policy setter does not reset `previous_selloff`, `current_selloff`, `window_start_timestamp`, or `selloff_vb_snapshot`. New settings affect subsequent swaps immediately, including an in-progress window. See [max-selloff window](../for-traders/max-selloff.md) and [fee mathematics](../technical/math.md) for the weighted two-window calculation and output charging.

## Range-manager configuration

| Field | Type and units | Meaning |
| --- | --- | --- |
| `range_manager` | `Pubkey` | Delegated signer; zero means unset |
| `range_manager_enabled` | `bool` | Independent permission gate |
| `range_manager_max_vb_change_pct` | `u16`, 10,000 = 100% | Maximum change relative to each old virtual balance per update |
| `range_manager_max_weight_change_pct` | `u16`, same scale | Maximum relative change of each old weight per update |
| `range_manager_min_update_interval_secs` | `u32`, seconds | Minimum interval between successful updates; zero removes time delay |
| `range_manager_last_updated` | `i64`, Unix seconds | Last successful manager update |
| `range_manager_max_leverage_bps` | `u32`, 10,000 = 1× | Optional maximum virtual/actual ratio on virtual-balance slots written by the update; zero disables ceiling |
| `range_manager_min_leverage_bps` | `u32`, same scale | Optional minimum ratio on the same slots; zero disables floor |

The two change caps must be at most 10,000; **zero means no nonzero change is allowed**, not unlimited change. The ratio bounds are not capped at 10,000 because leverage can exceed 1×. When both bounds are set, minimum must not exceed maximum. Zero-actual-balance slots are exempt from ratio bounds. See [controls](../safety/pool-controls.md#range-manager).

## Extension policy defaults

Fresh configs get `banned_extensions = 100684810` (bits 1, 3, 10, 12, 14, 25, 26) and `hard_banned_extensions = 512` (bit 9). Existing configs retain their stored values until explicitly changed. At pool creation:

```text
hard_floor = config.hard_banned_extensions == 0 ? 512 : config.hard_banned_extensions
effective = (creator_override ?? config.banned_extensions) OR hard_floor
```

A zero stored hard floor therefore uses the compile-time fallback; it does not mean there are no restrictions. NonTransferable and other exact unsafe creation conditions are also rejected independently of bitmaps. Lowering a creation bitmap does not make a mint compatible with every transfer instruction. See [token extension policy](../safety/pool-controls.md#token-extension-policy).
