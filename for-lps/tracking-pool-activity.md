# Tracking Pool Activity

This page describes how to monitor a Cube pool's state, performance, and activity.

---

## On-Chain Pool State

### Reading Pool Data

Use the `get_pool_info` instruction with transaction simulation to read all pool state without modifying it:

```typescript
const tx = await program.methods.getPoolInfo()
  .accounts({ pool: poolPubkey })
  .simulate();
// Parse PoolInfo event from tx.events
```

The `PoolInfo` event contains:

| Field | Type | Description |
|---|---|---|
| `pool` | `Pubkey` | Pool address |
| `config` | `Pubkey` | Config address |
| `bpt_mint` | `Pubkey` | BPT mint address |
| `token_count` | `u8` | Number of tokens (2–9 via UI/SDK; 10 supported on-chain) |
| `pool_id` | `u64` | Pool identifier |
| `tokens` | `Vec<Pubkey>` | Token mint addresses |
| `token_vaults` | `Vec<Pubkey>` | Vault addresses (derived as ATA(pool, mint, token_program)) |
| `normalized_weights` | `Vec<u64>` | Weights in basis points (sum = 10000) |
| `virtual_balances` | `Vec<u64>` | Virtual balances |
| `actual_balances` | `Vec<u64>` | Actual balances |
| `protocol_fees_owed` | `Vec<u64>` | Uncollected protocol fees |
| `swap_fee_rate` | `u32` | Swap fee rate (hundredths of bps) |
| `protocol_fee_rate` | `u16` | Protocol fee share (bps) |
| `pool_enabled` | `bool` | Master kill switch |
| `swaps_enabled` | `bool` | Per-pool swap toggle |
| `token_programs` | `Vec<Pubkey>` | Per-token program IDs (SPL or Token-2022) |

### Direct Account Deserialization

The `CubicPool` account is **1683 bytes (v4 AoS layout)**. Older v3 pools (1154 bytes, parallel arrays) are migrated via `migrate_pool_v4` — read the in-tree `programs/cubic-pool/src/instructions/migrate_pool_v4.rs` if you encounter one.

**Top-level layout (1683 bytes):**

| Offset | Field | Size | Type |
|---|---|---|---|
| 0–7 | discriminator | 8 | `[u8; 8]` |
| 8–39 | `config` | 32 | `Pubkey` |
| 40 | `bump` | 1 | `u8` |
| 41 | `token_count` | 1 | `u8` |
| 42–49 | `pool_id` | 8 | `u64` |
| 50–53 | `swap_fee_rate` | 4 | `u32` (hundredths of bps) |
| 54–55 | `protocol_fee_rate` | 2 | `u16` (bps) |
| 56–63 | `created_at` | 8 | `i64` (unix sec) |
| 64 | `pool_enabled` | 1 | `bool` (master kill — protocol-admin only) |
| 65 | `swaps_enabled` | 1 | `bool` (per-pool toggle) |
| 66–97 | `pool_admin` | 32 | `Pubkey` |
| 98–129 | `pending_pool_admin` | 32 | `Pubkey` (`default` = no transfer pending) |
| 130–161 | `range_manager` | 32 | `Pubkey` (`default` = no manager set) |
| 162 | `range_manager_enabled` | 1 | `bool` |
| 163–164 | `range_manager_max_vb_change_bps` | 2 | `u16` (≤ 10 000) |
| 165–166 | `range_manager_max_weight_change_bps` | 2 | `u16` (≤ 10 000) |
| 167–170 | `range_manager_min_update_interval_secs` | 4 | `u32` |
| 171–178 | `range_manager_last_updated` | 8 | `i64` |
| 179–1618 | `tokens` | 1440 | `[TokenSlot; 10]` (see below) |
| 1619–1650 | `lookup_table` | 32 | `Pubkey` (`default` = ALT not provisioned) |
| 1651–1682 | `reserved` | 32 | `[u8; 32]` |

**`TokenSlot` is 144 bytes**, packed as `config: AssetConfig (88) + dynamics: AssetDynamics (56)`. Slots past `token_count` are zero-padded — ignore them.

**`AssetConfig` (88 bytes) — admin-controlled, rarely changes:**

| Offset (relative) | Field | Size | Type |
|---|---|---|---|
| 0–31 | `mint` | 32 | `Pubkey` |
| 32–63 | `token_program` | 32 | `Pubkey` |
| 64–71 | `normalized_weight` | 8 | `u64` (basis points) |
| 72–79 | `max_selloff` | 8 | `u64` (raw token units, `0` = disabled — see [Max-Selloff Window](../for-traders/max-selloff.md)) |
| 80–83 | `max_selloff_period_length` | 4 | `u32` (seconds) |
| 84–87 | `reserved` | 4 | `[u8; 4]` |

**`AssetDynamics` (56 bytes) — auto-updated by user txs:**

| Offset (relative) | Field | Size | Type |
|---|---|---|---|
| 0–7 | `virtual_balance` | 8 | `u64` (raw units) |
| 8–15 | `actual_balance` | 8 | `u64` (raw units, mirrors vault) |
| 16–23 | `protocol_fees_owed` | 8 | `u64` (uncollected, awaits `collect_protocol_fees`) |
| 24–31 | `previous_selloff` | 8 | `u64` |
| 32–39 | `current_selloff` | 8 | `u64` |
| 40–47 | `window_start_timestamp` | 8 | `i64` |
| 48–55 | `reserved` | 8 | `[u8; 8]` |

For the i-th token (`0 ≤ i < token_count`):

```
slot_offset  = 179 + i * 144
config_off   = slot_offset
dynamics_off = slot_offset + 88
```

The struct definitions live in `programs/cubic-pool/src/state/cubic_pool.rs` — that file is the source of truth, this table is mechanically derived from it.

---

## Backend API Monitoring

> 🔐 **API access requires a key.** The endpoints below are gated. To get a key, reach out:
>
> - Telegram chat: **[@cubee\_chat](https://t.me/cubee_chat)**
> - Direct: **[@sepezho](https://t.me/sepezho)**

The Cube backend API is available at `https://api.cubee.ee`.

### Pool Details

```
GET https://api.cubee.ee/api/pools/:address
```

Returns the pool with all tokens, balances, weights, fees, TVL, **virtual TVL**, volume, APY, range-manager config, and per-token `max_selloff` settings.

### Pool Transactions

```
GET https://api.cubee.ee/api/pools/:address/transactions?limit=50&offset=0&user=<wallet>
```

Returns recent transactions (swaps, adds, removes) with full event data. Filterable by user wallet.

### Transaction Stats

```
GET https://api.cubee.ee/api/pools/:address/tx-stats
```

Returns `{ totalCount, count24h }` — total and 24-hour transaction counts.

### Platform Stats

```
GET https://api.cubee.ee/api/pools/stats
```

Returns `{ totalTvlUsd, totalVirtualTvl, poolCount, updatedAt }`.

See the full [API Reference](../integration/api-reference.md) for every endpoint, request format, and response shape.

---

## Key Metrics

### Spot Price

The instantaneous price between two tokens, using the contract's
`WeightedMath::calc_spot_price` formula. **This is NOT just
`vbIn / vbOut`** — that ignores weights and is wrong for non-50/50
pools.

```
spotPrice(in → out) = (vbIn / wIn) / (vbOut / wOut)
                    × 10^(decimalsOut − decimalsIn)
```

Where `vbIn`/`vbOut` are `virtual_balance` (raw `u64`), `wIn`/`wOut`
are `normalized_weight` (bps, sum = 10 000). The decimal correction
converts the raw-unit ratio into human-readable "out per 1 in".

Reference: `programs/cubic-pool/src/math/mod.rs` → `WeightedMath::calc_spot_price`. The SDK exposes this as `CubicPoolClient.quoteSwap(...)` — prefer the SDK over re-implementing.

### Leverage Ratio

Per-token leverage, showing how much "depth concentration" is applied:

```
leverage[i] = virtual_balance[i] / actual_balance[i]
```

`leverage = 1.0` means the AMM curve treats the pool exactly as the
vault holds. `leverage > 1` widens the quote (lower slippage for the
same actual depth). `leverage < 1` is allowed but tightens the curve
(very high slippage per dollar of actual reserves).

If leverage drifts significantly from the initial setting, it
indicates large unidirectional trading that hasn't been rebalanced
by the range-manager.

### BPT Value

The value of your LP position:

```
your_value = (your_bpt / bpt_total_supply) × sum(actual_balance[i] × token_price[i])
```

BPT supply and pool balances can be read on-chain or from the backend API.

### TVL (Total Value Locked)

The USD value of all actual balances in the pool. Updated periodically by the backend using prices from Pyth (where available) and pool spot prices (for tokens without a Pyth feed).

### Virtual TVL

`Σ (virtual_balance[i] × token_price[i])`. Differs from TVL by per-token leverage; surfaced separately so LPs can see the **AMM-curve depth**, not just the underlying asset value. Available via the `virtualTvlUsd` field on the pool API response (and the stats panel on the pool page).

### Volume (24h)

Trading volume over the last 24 hours, derived from on-chain transaction data indexed by the backend.

### APY

Annualised yield based on recent fee revenue:

```
daily_yield = (volume_24h × swap_fee_rate × (1 − protocol_fee_rate)) / tvl
apy = (1 + daily_yield) ^ 365 − 1
```

---

## Pool Status Flags

| Flag | When `false` | Effect |
|---|---|---|
| `pool_enabled` | Pool disabled | All operations blocked. Only `protocol_admin` can flip. |
| `swaps_enabled` | Swaps paused | Only swaps blocked; add/remove still work. Either `pool_admin` or `protocol_admin` can flip. |
| `range_manager_enabled` | Range-manager disabled | `range_manager_update` reverts even if the pubkey is set. Pool-admin only. |

All readable on-chain and returned by the backend API.

---

## Events for Indexing

Every pool operation emits a `PoolStateLog` event containing the full pool state snapshot. The backend transaction parser monitors these events to maintain an accurate off-chain index of pool state.

| Event | Trigger |
|---|---|
| `Swap` | Every swap |
| `LiquidityAdded` | Every deposit |
| `LiquidityRemoved` | Every withdrawal |
| `PoolStateLog` | Every operation (companion event) |
| `SwapFeeRateUpdated` | `set_swap_fee_rate` |
| `ProtocolFeeRateUpdated` | `set_protocol_fee_rate` |
| `PoolEnabledUpdated` | `set_pool_enabled` |
| `SwapsEnabledUpdated` | `set_swaps_enabled` |
| `MaxSelloffSet` | `set_max_selloff` |
| `RangeManagerSet` | `set_range_manager` |
| `RangeManagerConfigSet` | `set_range_manager_config` |
| `RangeManagerUpdated` | `range_manager_update` |
| `MaxSelloffWindowAdvanced` | Companion to `Swap` when a max-selloff check passed (sliding-window state shifted) |
| `ProtocolFeesCollected` | `collect_protocol_fees` |
