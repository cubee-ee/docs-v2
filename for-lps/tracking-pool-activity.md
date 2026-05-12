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
| `token_count` | `u8` | Number of tokens (2–10) |
| `pool_id` | `u64` | Pool identifier |
| `tokens` | `Vec<Pubkey>` | Token mint addresses |
| `token_vaults` | `Vec<Pubkey>` | Vault addresses (derived) |
| `normalized_weights` | `Vec<u64>` | Weights in basis points |
| `virtual_balances` | `Vec<u64>` | Virtual balances |
| `actual_balances` | `Vec<u64>` | Actual balances |
| `protocol_fees_owed` | `Vec<u64>` | Uncollected protocol fees |
| `swap_fee_rate` | `u32` | Swap fee rate |
| `protocol_fee_rate` | `u16` | Protocol fee rate |
| `pool_enabled` | `bool` | Pool enabled flag |
| `swaps_enabled` | `bool` | Swaps enabled flag |
| `token_programs` | `Vec<Pubkey>` | Per-token program IDs (SPL or Token-2022) |

### Direct Account Deserialization

You can also read the `CubicPool` account directly (1,154 bytes):

| Offset | Field | Size | Type |
|---|---|---|---|
| 0–7 | Discriminator | 8 | `[u8; 8]` |
| 8–39 | `config` | 32 | `Pubkey` |
| 40 | `bump` | 1 | `u8` |
| 41 | `token_count` | 1 | `u8` |
| 42–49 | `pool_id` | 8 | `u64` |
| 50–369 | `token_mints` | 320 | `[Pubkey; 10]` |
| 370–689 | `token_programs` | 320 | `[Pubkey; 10]` |
| 690–769 | `normalized_weights` | 80 | `[u64; 10]` |
| 770–849 | `virtual_balances` | 80 | `[u64; 10]` |
| 850–929 | `actual_balances` | 80 | `[u64; 10]` |
| 930–933 | `swap_fee_rate` | 4 | `u32` |
| 934–935 | `protocol_fee_rate` | 2 | `u16` |
| 936–1015 | `protocol_fees_owed` | 80 | `[u64; 10]` |
| 1016–1023 | `created_at` | 8 | `i64` |
| 1024 | `pool_enabled` | 1 | `bool` |
| 1025 | `swaps_enabled` | 1 | `bool` |
| 1026–1153 | `reserved` | 128 | `[u8; 128]` |

---

## Backend API Monitoring

The Cube backend API is available at `https://api.cubee.ee`.

### Pool Details

```
GET https://api.cubee.ee/api/pools/:address
```

Returns the pool with all tokens, balances, weights, fees, TVL, volume, and APY.

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

---

## Key Metrics

### Spot Price

The instantaneous price between two tokens, calculated from virtual balances:

```
spotPrice = (virtualBalanceIn * weightOut) / (virtualBalanceOut * weightIn)
```

### Leverage Ratio

Per-token leverage, showing how much concentration is applied:

```
leverage[i] = virtual_balance[i] / actual_balance[i]
```

If leverage drifts significantly from the initial setting, it indicates large unidirectional trading.

### BPT Value

The value of your LP position:

```
your_value = sum(actual_balance[i] * token_price[i]) * (your_bpt / bpt_total_supply)
```

BPT supply and pool balances can be read on-chain or from the backend API.

### TVL (Total Value Locked)

The USD value of all actual balances in the pool. Updated every 10 minutes by the backend using SOL price from Pyth oracle.

### Volume (24h)

Trading volume over the last 24 hours, derived from on-chain transaction data indexed by the backend.

### APY

Annualized yield based on recent fee revenue:

```
daily_yield = (volume_24h * swap_fee_rate * (1 - protocol_fee_rate)) / tvl
apy = (1 + daily_yield) ^ 365 - 1
```

---

## Pool Status Flags

| Flag | When `false` | Effect |
|---|---|---|
| `pool_enabled` | Pool disabled | All operations blocked |
| `swaps_enabled` | Swaps paused | Only swaps blocked; add/remove still work |

Both flags are readable on-chain and returned by the backend API.

---

## Events for Indexing

Every pool operation emits a `PoolStateLog` event containing the full pool state snapshot. The backend transaction parser monitors these events to maintain an accurate off-chain index of pool state.

| Event | Trigger |
|---|---|
| `Swap` | Every swap |
| `LiquidityAdded` | Every deposit |
| `LiquidityRemoved` | Every withdrawal |
| `PoolStateLog` | Every operation (companion event) |
| `SwapFeeRateUpdated` | Fee change |
| `ProtocolFeeRateUpdated` | Protocol fee change |
| `PoolEnabledUpdated` | Pool toggle |
| `SwapsEnabledUpdated` | Swaps toggle |
| `ProtocolFeesCollected` | Fee collection |
