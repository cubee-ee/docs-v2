# Accounts and Events

This reference is generated from the matching contract and SDK IDLs at contracts [`96a2ee2`](https://github.com/coffer-so/contracts/tree/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs) and SDK 0.11.1 [`09cc776`](https://github.com/coffer-so/sdk/tree/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/idl), with field meanings checked against the executable handlers. Fields appear in **Borsh serialization order**, with no Rust memory-alignment padding.

## Account identities and layouts

| Account | Owner | Address/size |
| --- | --- | --- |
| `CubicPool` | Cubic Pool program | PDA `["cubic_pool", config, pool_id_le]`; 1,683 bytes including discriminator |
| `CubicPoolConfig` | Cubic Pool program | Explicit config keypair at creation; 202 bytes including discriminator |
| `Treasury` | Protocol Admin program | PDA `["treasury"]`; current 818 bytes including discriminator |
| Pool BPT mint | Selected SPL Token or Token-2022 program | PDA `["bpt_mint", pool]` under Cubic Pool; 9 decimals and pool as mint authority |
| Pool asset vault | Token program of that asset | ATA derived with pool, mint and the slot’s stored token program |
| STLD helper | System program, dataless | PDA `["stld_helper", pool]` under Single Token Liquidity; signs CPIs, no Anchor state account |
| Treasury registered vault | Classic SPL Token | PDA `["vault", mint]` under Protocol Admin; token-account owner authority is Treasury |

STLD's IDL includes `CubicPool` and `CubicPoolConfig` because it consumes those account types; it does not own separate copies. BPT supply/owner/decimals and token mint decimals are read from their token-program accounts, not stored inside `CubicPool`.

Current pool and config byte sizes remain unchanged from the 1,683-byte v4 family, but reserved bytes and some token configuration fields gained meaning. There is no account-level version tag that proves the migration completed. Older 1,154-byte v3 pools are not handled by `migrate_to_v5`. A legacy 786-byte Treasury requires the supervisor realloc path before using the current full layout decoder. See [migration](smart-contracts.md#migration-compatibility).

The SDK's `decodeContractAccount(program, accountName, data)` checks current exact size and discriminator, returns IDL snake_case fields, and preserves u64/i64 values as `BN`. The caller must separately validate the RPC account's program owner. `CubicPoolClient.sync()` combines the stored pool with mint/BPT metadata and chain time for quoting. Neither decoding nor transaction simulation proves later execution will observe the same state.

The [state and quote guide](../sdk/state-and-quotes.md) maps every field below to
`RawPoolAccount` and the `PoolInfo`/`PoolTokenInfo` objects returned by `sync()`.
It explains which values are fetched from mint/Clock accounts, which are derived,
and which must be read separately. The current raw pool decoder also checks the
discriminator and retains signed i64 timestamps and the reserved tail.

### CubicPool

8-byte discriminator: `89d22a16d19c2b4e`. Offsets below are absolute within account data.

| Offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 8 | `config` | `Pubkey` | Parent config and part of the pool PDA seeds. |
| 40 | `bump` | `u8` | Stored PDA bump. |
| 41 | `token_count` | `u8` | Number of meaningful token slots, 2–10 for a pool; Treasury uses 0–10 registered entries. Read only this prefix of fixed arrays. |
| 42 | `pool_id` | `u64` | Caller-selected u64 seed; little endian in PDA derivation. |
| 50 | `swap_fee_rate` | `u32` | Base input fee, denominator 1,000,000, maximum 100,000. |
| 54 | `protocol_fee_rate` | `u16` | Protocol share of base input fee, denominator 10,000, maximum 5,000. |
| 56 | `created_at` | `i64` | Creation time in Unix seconds. |
| 64 | `pool_enabled` | `bool` | Full operational gate: swaps, LP operations, STLD and range-manager updates require true. |
| 65 | `swaps_enabled` | `bool` | Trading gate: direct swaps and STLD require true. |
| 66 | `pool_admin` | `Pubkey` | Per-pool authority, initially creator; zero means renounced. |
| 98 | `pending_pool_admin` | `Pubkey` | Proposed successor; zero means no pending transfer. |
| 130 | `range_manager` | `Pubkey` | Delegated updater pubkey; nonzero and enabled required to act. |
| 162 | `range_manager_enabled` | `bool` | Range-manager permission gate, independent of pool-admin renunciation. |
| 163 | `range_manager_max_vb_change_pct` | `u16` | Maximum relative virtual-balance change per call, denominator 10,000. Zero permits no nonzero change. |
| 165 | `range_manager_max_weight_change_pct` | `u16` | Maximum relative weight change per call, denominator 10,000. Zero permits no nonzero change. |
| 167 | `range_manager_min_update_interval_secs` | `u32` | Seconds between successful manager updates; zero removes time delay. |
| 171 | `range_manager_last_updated` | `i64` | Unix seconds of last successful manager update. |
| 179 | `tokens` | `[TokenSlot; 10]` | Ten fixed TokenSlot structures; active prefix is determined by token_count. |
| 1619 | `lookup_table` | `Pubkey` | Frozen pool ALT address; zero means none provisioned. |
| 1651 | `banned_extensions` | `u64` | On pool: effective creation-policy snapshot (or migration backfill). On config: default policy for future pools. Bit N corresponds to extension type N. |
| 1659 | `range_manager_max_leverage_bps` | `u32` | Maximum allowed new virtual/actual ratio for live VB slots written; 10,000 = 1×, zero disables ceiling. |
| 1663 | `range_manager_min_leverage_bps` | `u32` | Minimum allowed new virtual/actual ratio for the same slots; 10,000 = 1×, zero disables floor. |
| 1667 | `reserved` | `[u8; 16]` | Reserved bytes; no public operational meaning. Do not assume a separate version tag. |

Serialized account size: **1683 bytes**.

### CubicPoolConfig

8-byte discriminator: `041df30a8e40e876`. Offsets below are absolute within account data.

| Offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 8 | `protocol_admin` | `Pubkey` | Config-level authority, initially Treasury PDA; can rotate independently. |
| 40 | `pending_protocol_admin` | `Pubkey` | Pending config authority; zero means no pending transfer. |
| 72 | `default_protocol_fee_rate` | `u16` | Explicitly chosen config-init fee-share default, copied to newly created pools. |
| 74 | `banned_extensions` | `u64` | On pool: effective creation-policy snapshot (or migration backfill). On config: default policy for future pools. Bit N corresponds to extension type N. |
| 82 | `hard_banned_extensions` | `u64` | Config creation-policy floor; zero uses compile-time 512 fallback when a new pool is initialized. |
| 90 | `reserved` | `[u8; 112]` | Reserved bytes; no public operational meaning. Do not assume a separate version tag. |

Serialized account size: **202 bytes**.

### AssetConfig

Nested struct; offsets below start at this struct’s first byte.

| Offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 0 | `mint` | `Pubkey` | Token mint address fixed at pool initialization. |
| 32 | `token_program` | `Pubkey` | Owner program pinned for this mint, classic SPL Token or Token-2022. |
| 64 | `normalized_weight` | `u64` | Weight in basis points; each 100–9,900 and sum over all pool token slots is 10,000. |
| 72 | `max_selloff_pct` | `u16` | Window sell cap as a fraction of virtual-balance snapshot, denominator 10,000. Zero disables cap and surge. |
| 74 | `max_selloff_period_length` | `u32` | Window length in seconds, positive for an enabled limiter. |
| 78 | `variable_fee_threshold_pct` | `u16` | Window fill at which surge begins, denominator 10,000. |
| 80 | `variable_fee_slope_low_pct` | `u16` | Output fee rate at threshold, denominator 10,000. |
| 82 | `variable_fee_slope_high_pct` | `u16` | Output fee rate at full fill, denominator 10,000; zero disables surge. |
| 84 | `is_active` | `bool` | Swap-input gate for this slot; does not disable output or proportional LP operations. |
| 85 | `variable_fee_slope_mid_pct` | `u16` | Output fee rate at kink, denominator 10,000; low ≤ mid ≤ high. |
| 87 | `variable_fee_kink_pct` | `u8` | Kink position in whole percent. Zero selects a single straight line; nonzero is strictly between threshold and full fill. |

Serialized struct size: **88 bytes**.

### AssetDynamics

Nested struct; offsets below start at this struct’s first byte.

| Offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 0 | `virtual_balance` | `u64` | Raw token balance used for curve pricing; distinct from actual reserve. |
| 8 | `actual_balance` | `u64` | Raw LP-owned reserve. Protocol fees are already excluded: do not subtract owed fees again. |
| 16 | `protocol_fees_owed` | `u64` | Raw protocol-owned token units co-located in the vault. Includes input base-fee share or output surge accrual. |
| 24 | `previous_selloff` | `u64` | Gross input units accumulated in preceding window. |
| 32 | `current_selloff` | `u64` | Gross input units accumulated in current window. |
| 40 | `window_start_timestamp` | `i64` | Current window start in Unix seconds. |
| 48 | `selloff_vb_snapshot` | `u64` | Raw virtual-balance snapshot that fixes the window cap. Zero means it will be captured on the next enabled check. |

Serialized struct size: **56 bytes**.

### TokenSlot

Nested struct; offsets below start at this struct’s first byte.

| Offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 0 | `config` | `AssetConfig` | Per-token AssetConfig, not the pool’s parent config address. |
| 88 | `dynamics` | `AssetDynamics` | Per-token runtime balances and window state. |

Serialized struct size: **144 bytes**.

### Treasury

8-byte discriminator: `eeef7bee5901a8fd`. Offsets below are absolute within account data.

| Offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 8 | `admin` | `Pubkey` | Treasury admin, the outer signer allowed to request full Treasury actions. |
| 40 | `pending_admin` | `Pubkey` | Pending Treasury successor; zero cannot accept. |
| 72 | `bump` | `u8` | Stored PDA bump. |
| 73 | `token_count` | `u8` | Number of meaningful token slots, 2–10 for a pool; Treasury uses 0–10 registered entries. Read only this prefix of fixed arrays. |
| 74 | `token_mints` | `[Pubkey; 10]` | Treasury registered classic SPL mint array, prefix token_count. |
| 394 | `token_vaults` | `[Pubkey; 10]` | Registered Treasury vault addresses, aligned with token_mints. These are vault PDAs, not pool ATAs. |
| 714 | `created_at` | `i64` | Creation time in Unix seconds. |
| 722 | `reserved` | `[u8; 64]` | Reserved bytes; no public operational meaning. Do not assume a separate version tag. |
| 786 | `supervisor` | `Pubkey` | Optional limited signer; zero means absent. Can freeze/unfreeze pools and disable/enable token inputs in this branch. |

Serialized account size: **818 bytes**.

## Balance semantics

`actual_balance` is LP capital and `protocol_fees_owed` is protocol capital. Under ordinary AMM operation, `vault.amount = actual_balance + protocol_fees_owed`. Direct donations to a vault or issuer-side actions can invalidate that equality without updating the pool. Fee collection clears owed counters without altering actual/virtual balances. Do not value LP shares against the raw vault amount without separating protocol fees. The accounting updates and output-denominated surge are described in [math](math.md).

## Event decoding and provenance

There are **60 current event schemas**: 30 Cubic Pool, 28 Protocol Admin, and 2 STLD. Anchor emits each as `Program data: <base64(discriminator || Borsh payload)>`. `decodeContractEvent(payload, program?)` decodes one payload; `parseContractEvents(logs, program?)` exposes all current IDL fields and keeps u64/i64 values as BN. `parseCubicPoolEvents` supplies the SDK's camelCase compatibility API.

Decode only successful transactions for canonical accounting. Verify `meta.err == null` and the emitting program using the invocation stack: event-shaped bytes in a log are not proof that the expected program emitted them. The optional parser program selector chooses an ABI; it does not authenticate the log. A Treasury wrapper emits outer governance events as well as events emitted by the inner Cubic Pool/STLD CPI. Use program, transaction signature and event/log position to avoid double-counting.

Most fields ending in `amount`, `amounts`, `balances`, `selloff`, `cap` or `snapshot` are raw per-mint units. Vector entries use pool token order. Do not sum amounts across tokens with different decimals. Timestamps are Unix seconds, rates retain their parameter scales, and boolean old/new fields describe transitions. Event field order and types below are complete; do not invent missing fields from instruction parameters.

### Important event limits

- `Swap.amount_out` is what the user receives **after surge**. `fee_amount` and `protocol_fee_amount` are in the input token; `surge_fee_amount` is in the output token. The without-SF ABI does not contain transfer-fee-in/out fields.
- `PoolInfo` is a read convenience event, not full account serialization. It omits admin, range-manager, input-active, selloff and surge configuration. Read current account data for those fields.
- `MaxSelloffSet` does not include mid-rate or kink fields, despite the handler updating them. `RangeManagerConfigSet` omits the minimum-leverage bound. To track these values fully, decode the successful instruction arguments and/or re-read the account.
- `ConfigInitialized` does not emit the hard-floor field. The outer `PoolBannedExtensionsSet` event also omits hard-mask fields; the inner `BannedExtensionsUpdated` contains old/new hard masks.
- `SingleTokenDeposit.deposited_amounts` is the capped basket passed into the proportional join; `LiquidityAdded.token_amounts` from the inner CPI is the amount actually transferred after that join's rounding. `dust_refunded` is an index-aligned vector. The helper forwards newly minted BPT only and may consume/refund pre-existing token dust.
- `PoolStateLog` contains balances, not complete governance or selloff configuration. `MaxSelloffWindowAdvanced` reports the input token's enabled window state after a successful swap. A disabled limiter does not produce that event.
- `pool_migrate_to_v5` has no separate Protocol Admin migration-event schema; observe the inner `PoolMigratedToV5` event.

## Complete event schemas

### cubic_pool (30 events)

#### BannedExtensionsUpdated

Discriminator: `6b7e0d95b66c8bca`.

| Field in order | Type |
| --- | --- |
| `config` | `Pubkey` |
| `authority` | `Pubkey` |
| `old_value` | `u64` |
| `new_value` | `u64` |
| `timestamp` | `i64` |
| `old_hard_value` | `u64` |
| `new_hard_value` | `u64` |

#### ConfigInitialized

Discriminator: `b531c89c13a7b25b`.

| Field in order | Type |
| --- | --- |
| `config` | `Pubkey` |
| `protocol_admin` | `Pubkey` |
| `default_protocol_fee_rate` | `u16` |
| `banned_extensions` | `u64` |
| `payer` | `Pubkey` |
| `timestamp` | `i64` |

#### DebugLiquidityWithdrawn

Discriminator: `ae3b951687818153`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `token_amounts` | `Vec<u64>` |
| `timestamp` | `i64` |

#### LiquidityAdded

Discriminator: `9a1add6cee40d9a1`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `user` | `Pubkey` |
| `token_amounts` | `Vec<u64>` |
| `bpt_amount` | `u64` |
| `timestamp` | `i64` |

#### LiquidityRemoved

Discriminator: `e169d8277c74a9bd`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `user` | `Pubkey` |
| `bpt_amount` | `u64` |
| `token_amounts` | `Vec<u64>` |
| `timestamp` | `i64` |

#### MaxSelloffSet

Discriminator: `86b73186528a55ea`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `max_selloff_pct` | `Vec<u16>` |
| `max_selloff_period_length` | `Vec<u32>` |
| `fee_threshold_pct` | `Vec<u16>` |
| `fee_slope_low_pct` | `Vec<u16>` |
| `fee_slope_high_pct` | `Vec<u16>` |
| `timestamp` | `i64` |
| `old_max_selloff_pct` | `Vec<u16>` |
| `old_max_selloff_period_length` | `Vec<u32>` |
| `old_fee_threshold_pct` | `Vec<u16>` |
| `old_fee_slope_low_pct` | `Vec<u16>` |
| `old_fee_slope_high_pct` | `Vec<u16>` |

#### MaxSelloffWindowAdvanced

Discriminator: `e5e3a31e16b74e39`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `token_index` | `u8` |
| `effective_selloff` | `u64` |
| `max_selloff_cap` | `u64` |
| `vb_snapshot` | `u64` |
| `previous_selloff` | `u64` |
| `current_selloff` | `u64` |
| `window_start_timestamp` | `i64` |
| `timestamp` | `i64` |

#### PoolAdminDisabled

Discriminator: `9e62ca25f339f757`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `old_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolAdminTransferCancelled

Discriminator: `ee35268fc22dceb3`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `admin` | `Pubkey` |
| `cancelled_pending` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolAdminTransferInitiated

Discriminator: `21c3ca85d12a9d4e`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `current_admin` | `Pubkey` |
| `pending_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolAdminTransferred

Discriminator: `7b30e7ba35439cfe`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `old_admin` | `Pubkey` |
| `new_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolAltInitialized

Discriminator: `3dd2561464fd2985`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `lookup_table` | `Pubkey` |
| `authority` | `Pubkey` |
| `address_count` | `u8` |
| `timestamp` | `i64` |

#### PoolEnabledUpdated

Discriminator: `652f03f0c5b5ec8e`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `old_value` | `bool` |
| `new_value` | `bool` |
| `timestamp` | `i64` |

#### PoolInfo

Discriminator: `cf145761fbd4ea2d`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `config` | `Pubkey` |
| `bpt_mint` | `Pubkey` |
| `token_count` | `u8` |
| `pool_id` | `u64` |
| `tokens` | `Vec<Pubkey>` |
| `token_vaults` | `Vec<Pubkey>` |
| `normalized_weights` | `Vec<u64>` |
| `virtual_balances` | `Vec<u64>` |
| `actual_balances` | `Vec<u64>` |
| `protocol_fees_owed` | `Vec<u64>` |
| `swap_fee_rate` | `u32` |
| `protocol_fee_rate` | `u16` |
| `pool_enabled` | `bool` |
| `swaps_enabled` | `bool` |
| `token_programs` | `Vec<Pubkey>` |
| `timestamp` | `i64` |

#### PoolInitialized

Discriminator: `6476ad570cc6fee5`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `config` | `Pubkey` |
| `token_count` | `u8` |
| `bpt_mint` | `Pubkey` |
| `timestamp` | `i64` |
| `banned_extensions` | `u64` |

#### PoolMigratedToV5

Discriminator: `5f379d3d5c064f69`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `token_count` | `u8` |
| `reactivated` | `u8` |
| `timestamp` | `i64` |

#### PoolSolWithdrawn

Discriminator: `52dcadaf18a5ace9`.

| Field in order | Type |
| --- | --- |
| `source` | `Pubkey` |
| `authority` | `Pubkey` |
| `recipient` | `Pubkey` |
| `amount` | `u64` |
| `timestamp` | `i64` |

#### PoolStateLog

Discriminator: `3bfeed6fa30a8ce0`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `virtual_balances` | `Vec<u64>` |
| `actual_balances` | `Vec<u64>` |
| `protocol_fees_owed` | `Vec<u64>` |
| `timestamp` | `i64` |

#### ProtocolAdminTransferCancelled

Discriminator: `2e78bb39883f5907`.

| Field in order | Type |
| --- | --- |
| `config` | `Pubkey` |
| `admin` | `Pubkey` |
| `cancelled_pending` | `Pubkey` |
| `timestamp` | `i64` |

#### ProtocolAdminTransferInitiated

Discriminator: `4ecfbc921a4128a7`.

| Field in order | Type |
| --- | --- |
| `config` | `Pubkey` |
| `current_admin` | `Pubkey` |
| `pending_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### ProtocolAdminTransferred

Discriminator: `8c1e3779694b9444`.

| Field in order | Type |
| --- | --- |
| `config` | `Pubkey` |
| `old_admin` | `Pubkey` |
| `new_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### ProtocolFeeRateUpdated

Discriminator: `bd380741005fc006`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `old_rate` | `u16` |
| `new_rate` | `u16` |
| `timestamp` | `i64` |
| `authority` | `Pubkey` |

#### ProtocolFeesCollected

Discriminator: `a5227d9b0f5663bf`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `token_amounts` | `Vec<u64>` |
| `timestamp` | `i64` |

#### RangeManagerConfigSet

Discriminator: `bb3b622f6c02e598`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `max_vb_change_pct` | `u16` |
| `max_weight_change_pct` | `u16` |
| `min_update_interval_secs` | `u32` |
| `timestamp` | `i64` |
| `old_max_vb_change_pct` | `u16` |
| `old_max_weight_change_pct` | `u16` |
| `old_min_update_interval_secs` | `u32` |
| `old_max_leverage_bps` | `u32` |
| `new_max_leverage_bps` | `u32` |

#### RangeManagerSet

Discriminator: `3e98eb5bc3ee53ca`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `old_manager` | `Pubkey` |
| `new_manager` | `Pubkey` |
| `enabled` | `bool` |
| `timestamp` | `i64` |
| `old_enabled` | `bool` |
| `new_enabled` | `bool` |

#### RangeManagerUpdated

Discriminator: `8490649095cdd418`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `manager` | `Pubkey` |
| `vb_indices` | `bytes` |
| `vb_new_values` | `Vec<u64>` |
| `weight_indices` | `bytes` |
| `weight_new_values` | `Vec<u64>` |
| `timestamp` | `i64` |

#### Swap

Discriminator: `516ce3becdd00ac4`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `user` | `Pubkey` |
| `token_in` | `Pubkey` |
| `token_out` | `Pubkey` |
| `amount_in` | `u64` |
| `amount_out` | `u64` |
| `fee_amount` | `u64` |
| `protocol_fee_amount` | `u64` |
| `timestamp` | `i64` |
| `surge_fee_amount` | `u64` |

#### SwapFeeRateUpdated

Discriminator: `658418ff5bfde365`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `old_rate` | `u32` |
| `new_rate` | `u32` |
| `timestamp` | `i64` |
| `authority` | `Pubkey` |

#### SwapsEnabledUpdated

Discriminator: `3774768a661ae3df`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `old_value` | `bool` |
| `new_value` | `bool` |
| `timestamp` | `i64` |

#### TokenActiveSet

Discriminator: `a8dd1992eae6e864`.

| Field in order | Type |
| --- | --- |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `token_index` | `u8` |
| `old_value` | `bool` |
| `new_value` | `bool` |
| `timestamp` | `i64` |

### protocol_admin (28 events)

#### AdminTransferCancelled

Discriminator: `5d174537d8806a38`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `admin` | `Pubkey` |
| `cancelled_pending` | `Pubkey` |
| `timestamp` | `i64` |

#### AdminTransferInitiated

Discriminator: `9dff005c346a109c`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `current_admin` | `Pubkey` |
| `pending_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### AdminTransferred

Discriminator: `ff93b605c7d926b3`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `old_admin` | `Pubkey` |
| `new_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### FundsWithdrawn

Discriminator: `3882e69a235c0b76`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `admin` | `Pubkey` |
| `mint` | `Pubkey` |
| `recipient` | `Pubkey` |
| `amount` | `u64` |
| `timestamp` | `i64` |

#### PoolAltInitializedByTreasury

Discriminator: `31959462df7f846f`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `pool` | `Pubkey` |
| `lookup_table` | `Pubkey` |
| `admin` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolBannedExtensionsSet

Discriminator: `27be9113ee33721f`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `config` | `Pubkey` |
| `banned_extensions` | `u64` |
| `timestamp` | `i64` |
| `old_banned_extensions` | `u64` |
| `new_banned_extensions` | `u64` |
| `admin` | `Pubkey` |

#### PoolConfigInitialized

Discriminator: `e5dc57f1afa5c61c`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `config` | `Pubkey` |
| `default_protocol_fee_rate` | `u16` |
| `timestamp` | `i64` |

#### PoolDebugWithdrawn

Discriminator: `f8ef428dcfa0947b`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `pool` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolEnabledSet

Discriminator: `b0645e8ac4891297`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `pool` | `Pubkey` |
| `enabled` | `bool` |
| `timestamp` | `i64` |
| `old_enabled` | `bool` |
| `new_enabled` | `bool` |
| `admin` | `Pubkey` |

#### PoolFeesCollected

Discriminator: `da4aa32a089839f2`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `pool` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolProtocolAdminTransferAccepted

Discriminator: `eb5b3f41ae2a3781`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `config` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolProtocolAdminTransferCancelled

Discriminator: `cea57071cb402597`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `config` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolProtocolAdminTransferInitiated

Discriminator: `258ef8847b875e16`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `config` | `Pubkey` |
| `new_admin` | `Pubkey` |
| `timestamp` | `i64` |

#### PoolProtocolFeeRateSet

Discriminator: `bd65f12631ae3117`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `pool` | `Pubkey` |
| `protocol_fee_rate` | `u16` |
| `timestamp` | `i64` |
| `old_rate` | `u16` |
| `new_rate` | `u16` |
| `admin` | `Pubkey` |

#### PoolSolWithdrawnViaTreasury

Discriminator: `af61e867f733719b`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `admin` | `Pubkey` |
| `source` | `Pubkey` |
| `recipient` | `Pubkey` |
| `amount` | `u64` |
| `timestamp` | `i64` |

#### PoolSwapsEnabledSet

Discriminator: `736fee98eb1ac6a8`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `pool` | `Pubkey` |
| `enabled` | `bool` |
| `timestamp` | `i64` |
| `old_enabled` | `bool` |
| `new_enabled` | `bool` |
| `admin` | `Pubkey` |

#### PoolTokenActiveSet

Discriminator: `5adf18e5f954852b`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `pool` | `Pubkey` |
| `authority` | `Pubkey` |
| `token_index` | `u8` |
| `is_active` | `bool` |
| `timestamp` | `i64` |
| `old_is_active` | `bool` |
| `new_is_active` | `bool` |

#### PoolsFrozen

Discriminator: `71780f5fdf17aca2`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `authority` | `Pubkey` |
| `pool_count` | `u32` |
| `timestamp` | `i64` |
| `pools` | `Vec<Pubkey>` |
| `old_enabled` | `Vec<bool>` |

#### PoolsUnfrozen

Discriminator: `7c23672d012a3e54`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `authority` | `Pubkey` |
| `pool_count` | `u32` |
| `timestamp` | `i64` |
| `pools` | `Vec<Pubkey>` |
| `old_enabled` | `Vec<bool>` |

#### ProgramClosed

Discriminator: `0369c3bb7a018e08`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `program` | `Pubkey` |
| `recipient` | `Pubkey` |
| `timestamp` | `i64` |

#### ProgramFrozen

Discriminator: `ad8035ef0f26ebcb`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `programdata` | `Pubkey` |
| `timestamp` | `i64` |

#### ProgramUpgraded

Discriminator: `7466dc04dcfad10d`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `program` | `Pubkey` |
| `timestamp` | `i64` |

#### SolWithdrawn

Discriminator: `91f94530ce565b42`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `admin` | `Pubkey` |
| `recipient` | `Pubkey` |
| `amount` | `u64` |
| `timestamp` | `i64` |

#### StldSolWithdrawnViaTreasury

Discriminator: `f7bf7f4772934f50`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `admin` | `Pubkey` |
| `source` | `Pubkey` |
| `recipient` | `Pubkey` |
| `amount` | `u64` |
| `timestamp` | `i64` |

#### SupervisorSet

Discriminator: `fafc43d78e346a3d`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `admin` | `Pubkey` |
| `old_supervisor` | `Pubkey` |
| `new_supervisor` | `Pubkey` |
| `timestamp` | `i64` |

#### TokenRegistered

Discriminator: `d226f9b64f70f0e1`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `mint` | `Pubkey` |
| `vault` | `Pubkey` |
| `token_index` | `u8` |
| `timestamp` | `i64` |

#### TreasuryInitialized

Discriminator: `c749aecd3b9137b3`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `admin` | `Pubkey` |
| `timestamp` | `i64` |

#### UpgradeAuthorityTransferred

Discriminator: `85c66b5a32b6c040`.

| Field in order | Type |
| --- | --- |
| `treasury` | `Pubkey` |
| `programdata` | `Pubkey` |
| `new_authority` | `Pubkey` |
| `timestamp` | `i64` |
| `old_authority` | `Pubkey` |

### single_token_liquidity (2 events)

#### SingleTokenDeposit

Discriminator: `d7368968db27a4eb`.

| Field in order | Type |
| --- | --- |
| `helper` | `Pubkey` |
| `pool` | `Pubkey` |
| `user` | `Pubkey` |
| `token_in_index` | `u8` |
| `amount_in` | `u64` |
| `allocations` | `Vec<u64>` |
| `deposited_amounts` | `Vec<u64>` |
| `bpt_received` | `u64` |
| `dust_refunded` | `Vec<u64>` |
| `timestamp` | `i64` |

#### StldSolWithdrawn

Discriminator: `18f61b27dec4873d`.

| Field in order | Type |
| --- | --- |
| `source` | `Pubkey` |
| `authority` | `Pubkey` |
| `recipient` | `Pubkey` |
| `amount` | `u64` |
| `timestamp` | `i64` |

## Error handling

Instruction failures include both custom program errors and propagated Anchor, token-program, loader or runtime errors. The [three IDLs](https://github.com/coffer-so/sdk/tree/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/idl) carry custom error codes and names. Decode the originating program rather than treating equal numeric codes from different programs as identical errors. Failed transactions can leave diagnostic logs but roll back their on-chain state changes; transaction fees may still be charged. Refresh state after stale-value, limit or slippage failures before constructing a replacement transaction.
