# Instruction Reference

This page covers all **59 instructions** in the three current contract IDLs: 28 Cubic Pool, 29 Protocol Admin, and 2 Single Token Liquidity. Verified against contracts [`96a2ee2`](https://github.com/coffer-so/contracts/tree/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs) and the matching SDK 0.11.1 IDLs at [`09cc776`](https://github.com/coffer-so/sdk/tree/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/idl).

## Encoding and units

Instruction names, argument names and account order below match the IDL exactly. Arguments are Borsh encoded after the 8-byte discriminator. `u64` token/BPT amounts use raw integer units; SOL amounts are lamports; `i64` timestamps are Unix seconds; `recent_slot` is a slot, not a timestamp. `pool_id` is a little-endian u64 seed. `N` means `pool.token_count`. Vectors of token amounts follow pool token order, not wallet balances or sorted mint order.

`swap_fee_rate` uses 1,000,000 = 100%; `protocol_fee_rate`, weights, most percentage parameters and leverage bounds use 10,000 = 100% or 1× as appropriate. `fee_kink_pct` alone uses whole percent. See [pool parameters](../overview/pool-parameters.md) for validation and [math](math.md) for rounding.

Each fixed account is listed in serialization order. **w** = writable, **s** = signer; unmarked = read-only nonsigner. Signer flags describe the instruction boundary: a PDA signer must be supplied by an authorized CPI, not a wallet signature. Account constraints in the handler remain necessary even when ABI encoding succeeds. Additional remaining accounts appear after all fixed accounts, in the layout explicitly stated for the operation.

CPI wrappers can emit an outer event as well as events from inner instructions. The per-instruction event list below names that handler’s own successful event; it is not a promise that no additional CPI event is present.

All these entrypoints return `Result<()>`: success/failure, with no typed instruction return payload. Read results from accounts and events. `get_pool_info` emits an event rather than return data. SDK `buildContractInstruction` exposes every instruction with exact snake_case keys; integer widths above 32 use `BN`, pubkeys use `PublicKey`, and absent `Option` values use `null`. The high-level [SDK clients](../sdk/index.md) add account derivation and validation.

## Structured arguments

### SelloffParams

| Field (serialization order) | Type |
| --- | --- |
| `max_selloff_pct` | `u16` |
| `period_length` | `u32` |
| `fee_threshold_pct` | `u16` |
| `fee_slope_low_pct` | `u16` |
| `fee_slope_high_pct` | `u16` |
| `fee_slope_mid_pct` | `u16` |
| `fee_kink_pct` | `u8` |

### TokenChange

| Field (serialization order) | Type | Meaning |
| --- | --- | --- |
| `index` | `u8` | Index below `token_count`; no duplicate index within either change list |
| `expected_current` | `u64` | Observed raw virtual balance in `vb_changes`, or weight in basis points in `weight_changes` |
| `new_value` | `u64` | Positive replacement in the same units; subject to the manager's per-update limits and final pool invariants |

`SelloffParams` replaces the full policy vector. `fee_slope_mid_pct` and `fee_kink_pct` come after `fee_slope_high_pct`; do not rearrange them into visual curve order. `TokenChange.expected_current` is the compare-and-swap guard. See [controls](../safety/pool-controls.md) for all associated validation.

## cubic_pool

### cubic_pool.accept_pool_admin_transfer

Discriminator: `9b2e7f5ef7052eb8`.

Replaces pool_admin with signer and clears pending successor.

**Authority:** Nonzero pending_pool_admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `new_admin` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolAdminTransferred`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/accept_pool_admin_transfer.rs).

### cubic_pool.accept_protocol_admin_transfer

Discriminator: `6d4655c3f427f466`.

Replaces config.protocol_admin and clears pending. A PDA successor must accept through its own program CPI; Treasury has pool_accept_protocol_admin_transfer.

**Authority:** Nonzero pending_protocol_admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` | w | — |
| `new_admin` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ProtocolAdminTransferred`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/accept_protocol_admin_transfer.rs).

### cubic_pool.add_liquidity

Discriminator: `b59d59438fb63448`.

Requires pool_enabled. Seed at zero BPT supply takes the supplied basket with at least one positive amount and mints from virtual-balance invariant; supply must be at least 1,000. Later amounts are ceilings for a proportional basket, live/zero-reserve token liveness must match. minimum_bpt_amount bounds minted BPT. Token accounts needed for nonzero transfers must already exist; the normal SDK transaction creates only the user BPT ATA. See [account setup](../for-lps/liquidity.md#account-setup). SDK additionally rejects zero-transfer live legs.

**Authority:** User; initial seed additionally requires non-renounced pool_admin.

**Arguments in order:** `token_amounts: Vec<u64>`, `minimum_bpt_amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `bpt_mint` | w | — |
| `user_bpt_account` | w | — |
| `user` | s | — |
| `token_program` |  | — |

**Remaining accounts:** Exactly 4N: `[user_token_0 (w), vault_0 (w), …, user_token_N−1 (w), vault_N−1 (w), mint_0 … mint_N−1, token_program_0 … token_program_N−1]`. **Pairs first, then all mints, then all programs.**

**Events:** `LiquidityAdded`, `PoolStateLog`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/add_liquidity.rs).

### cubic_pool.cancel_pool_admin_transfer

Discriminator: `9787cd7d5da0878f`.

Clears pending_pool_admin; does not change current admin.

**Authority:** Non-renounced current pool_admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolAdminTransferCancelled`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/cancel_pool_admin_transfer.rs).

### cubic_pool.cancel_protocol_admin_transfer

Discriminator: `b71a8206832420ba`.

Clears the pending config protocol authority.

**Authority:** Nonzero current config.protocol_admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ProtocolAdminTransferCancelled`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/cancel_protocol_admin_transfer.rs).

### cubic_pool.collect_protocol_fees

Discriminator: `1643176296b246dc`.

Moves each positive protocol_fees_owed amount to the supplied matching-mint/program recipient and zeros only that counter. Does not reduce actual or virtual balances. Recipient wallet unrestricted by the pool program. Zero-owed slots skip transfer/recipient checks.

**Authority:** Nonzero config.protocol_admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** Exactly 3N: `[vault_i (w), recipient_i (w), token_program_i]` for each token in pool order.

**Events:** `ProtocolFeesCollected`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/collect_protocol_fees.rs).

### cubic_pool.debug_withdraw_liquidity

Discriminator: `8b287e67c7228f89`.

Requires pool_enabled=false. Transfers the specified raw basket from canonical vaults; zeros skip. Does not reconcile actual/virtual balances or BPT supply; keep the affected pool retired. This is not normal fee collection.

**Authority:** Nonzero config.protocol_admin.

**Arguments in order:** `token_amounts: Vec<u64>`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** Exactly 3N: `[vault_i (w), recipient_i (w), token_program_i]` for each token in pool order.

**Events:** `DebugLiquidityWithdrawn`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/debug_withdraw_liquidity.rs).

### cubic_pool.disable_pool_admin

Discriminator: `6a6a70ff5c1f028a`.

Permanently clears pool_admin and pending_pool_admin. Leaves existing range manager and protocol authority intact; does not establish full pool immutability.

**Authority:** Non-renounced current pool_admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolAdminDisabled`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/disable_pool_admin.rs).

### cubic_pool.get_pool_info

Discriminator: `0930dc6516f04ec8`.

Emits PoolInfo; does not mutate the pool or return instruction data. Simulate to read without paying for a transaction. The event omits admin, range-manager, selloff and surge-policy fields; use account decoding for full state.

**Authority:** No program-level signer required.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolInfo`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/get_pool_info.rs).

### cubic_pool.initialize_config

Discriminator: `d07f1501c2bec446`.

Creates a 202-byte config; Treasury PDA is enforced as a signer and becomes protocol_admin. default_protocol_fee_rate ≤ 5,000; defaults initialize extension policy. Normal outer path: protocol_admin.pool_initialize_config.

**Authority:** Treasury PDA CPI, config keypair and payer signatures.

**Arguments in order:** `default_protocol_fee_rate: u16`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` | w, s | — |
| `protocol_admin_treasury` | s | — |
| `payer` | w, s | — |
| `system_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ConfigInitialized`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/initialize_config.rs).

### cubic_pool.initialize_cubic_pool

Discriminator: `d79474cf79686f83`.

Creates pool and BPT mint under an existing config, with no reserve vault ATA creation. 2–10 distinct mint accounts define token order. Weights sum to 10,000, each 100–9,900; virtual balances > 0; fee ≤ 100,000; decimals ≤ 18; extension checks apply. Starts with zero actual balances; seed deposit is separate.

**Authority:** Any payer; becomes pool_admin.

**Arguments in order:** `normalized_weights: Vec<u64>`, `initial_virtual_balances: Vec<u64>`, `swap_fee_rate: u32`, `pool_id: u64`, `banned_extensions_override: Option<u64>`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `bpt_mint` | w | — |
| `payer` | w, s | — |
| `token_program` |  | — |
| `associated_token_program` |  | — |
| `system_program` |  | — |

**Remaining accounts:** Exactly N read-only mint accounts, in token order. Their addresses define the token set.

**Events:** `PoolInitialized`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/initialize_cubic_pool.rs).

### cubic_pool.initialize_pool_alt

Discriminator: `fb874202f4490c90`.

Requires non-renounced pool_admin even on protocol path and unset lookup_table. Creates, extends with 6 + 2×token_count pool addresses, then freezes ALT; stores address on pool. Address derives from authority and recent_slot. Repeated init fails. The table contains pool-scoped addresses, not user/helper ATAs; it creates no token accounts. Newly added addresses cannot be used in the same slot.

**Authority:** Pool admin OR config.protocol_admin; separate payer.

**Arguments in order:** `recent_slot: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `config` |  | — |
| `authority` | s | — |
| `payer` | w, s | — |
| `lookup_table` | w | — |
| `system_program` |  | — |
| `alt_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolAltInitialized`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/initialize_pool_alt.rs).

### cubic_pool.initiate_pool_admin_transfer

Discriminator: `0a94936249be1f2b`.

Stores a nonzero pending successor; current admin retains power until acceptance. Does not auto-disable an existing range manager.

**Authority:** Non-renounced current pool_admin.

**Arguments in order:** `new_admin: Pubkey`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolAdminTransferInitiated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/initiate_pool_admin_transfer.rs).

### cubic_pool.initiate_protocol_admin_transfer

Discriminator: `a78127b60a6a3394`.

Stores a nonzero successor; it need not equal Treasury PDA. This is config authority rotation, not Treasury admin or program upgrade-authority rotation.

**Authority:** Nonzero current config.protocol_admin.

**Arguments in order:** `new_admin: Pubkey`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ProtocolAdminTransferInitiated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/initiate_protocol_admin_transfer.rs).

### cubic_pool.migrate_to_v5

Discriminator: `7f3c87ccb55acaf9`.

Only current 1,683-byte layout. Backfills a zero pool banned_extensions from config; reactivate_tokens=true activates all token inputs. No realloc, balance rewrite or v3 (1,154-byte) conversion. True can undo later deliberate token disables; it is not a harmless generic retry.

**Authority:** Nonzero pool_admin OR config.protocol_admin.

**Arguments in order:** `reactivate_tokens: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `config` |  | — |
| `authority` | s | — |
| `system_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolMigratedToV5`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/migrate_to_v5.rs).

### cubic_pool.range_manager_update

Discriminator: `f5b72345c80aac2c`.

Requires pool_enabled, interval, nonempty sparse changes, unique valid indices and matching expected_current. Enforces positive values, relative caps, final weight invariant, and absolute ratio bounds for nonzero-reserve VB slots written. Updates last_updated. No token transfer.

**Authority:** Nonzero configured enabled range_manager.

**Arguments in order:** `vb_changes: Vec<TokenChange>`, `weight_changes: Vec<TokenChange>`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `RangeManagerUpdated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/range_manager_update.rs).

### cubic_pool.remove_liquidity

Discriminator: `5055d14818ceb16c`.

Requires pool_enabled and positive requested burn ≤ supply. Effective burn is min(request, supply − 1,000); excess BPT remains in the wallet. Pays proportional actual reserves with fixed-point rounding and checks every minimum_token_amounts entry against the effective burn. Every user reserve-token account is validated even for a zero output; the normal SDK transaction creates these ATAs idempotently before removal.

**Authority:** BPT account owner (user).

**Arguments in order:** `bpt_amount: u64`, `minimum_token_amounts: Vec<u64>`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `bpt_mint` | w | — |
| `user_bpt_account` | w | — |
| `user` | s | — |
| `token_program` |  | — |

**Remaining accounts:** Exactly 4N: `[vault_0 (w), user_token_0 (w), …, vault_N−1 (w), user_token_N−1 (w), mint_0 … mint_N−1, token_program_0 … token_program_N−1]`. Vault/user order is reversed relative to add.

**Events:** `LiquidityRemoved`, `PoolStateLog`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/remove_liquidity.rs).

### cubic_pool.set_banned_extensions

Discriminator: `565ff9958e696497`.

Writes default and hard-floor u64 extension masks on a config. Only future pool creation consumes the new values; existing pool snapshots are unchanged. A zero hard value uses the compile-time 512 fallback during creation.

**Authority:** Nonzero config.protocol_admin.

**Arguments in order:** `banned_extensions: u64`, `hard_banned_extensions: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `BannedExtensionsUpdated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/set_banned_extensions.rs).

### cubic_pool.set_max_selloff

Discriminator: `642c44c6213a93fe`.

Replaces all token policies with exactly token_count SelloffParams entries. Checks cap, period and monotonic curve/kink bounds; keeps window accumulators and snapshot unchanged. See the detailed struct and pool controls.

**Authority:** Non-renounced pool_admin.

**Arguments in order:** `params: Vec<SelloffParams>`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `MaxSelloffSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_max_selloff.rs).

### cubic_pool.set_pool_enabled

Discriminator: `354caa2537de3f15`.

Sets the full operating gate: swaps, add/remove, STLD and range-manager updates require true. Administrative/recovery paths remain reachable under their own constraints.

**Authority:** Nonzero config.protocol_admin.

**Arguments in order:** `enabled: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolEnabledUpdated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/set_pool_enabled.rs).

### cubic_pool.set_protocol_fee_rate

Discriminator: `5f0704329a4f9c83`.

Sets protocol share of base fee on 10,000 scale, at most 5,000. Does not set surge rate or change another pool/default.

**Authority:** Nonzero config.protocol_admin.

**Arguments in order:** `protocol_fee_rate: u16`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ProtocolFeeRateUpdated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/set_protocol_fee_rate.rs).

### cubic_pool.set_range_manager

Discriminator: `6766ad7008022584`.

Pool admin may appoint/rotate or enable/disable. Protocol authority may only pass enabled=false and the existing manager pubkey. Current Treasury program has no wrapper for this instruction; its full-pool freeze remains available.

**Authority:** Pool admin; config protocol authority has disable-only path.

**Arguments in order:** `new_manager: Pubkey`, `enabled: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `RangeManagerSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_range_manager.rs).

### cubic_pool.set_range_manager_config

Discriminator: `67d300c8a4bdf5cd`.

Sets relative change caps ≤ 10,000, minimum interval, and absolute maximum/minimum VB/actual ratio bounds. Zero relative cap disallows changes; zero absolute bound disables that side. If both absolute bounds are set, min ≤ max.

**Authority:** Non-renounced pool_admin.

**Arguments in order:** `max_vb_change_pct: u16`, `max_weight_change_pct: u16`, `min_update_interval_secs: u32`, `max_leverage_bps: u32`, `min_leverage_bps: u32`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `RangeManagerConfigSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_range_manager_config.rs).

### cubic_pool.set_swap_fee_rate

Discriminator: `906651af873224bf`.

Sets base fee on 1,000,000 scale, at most 100,000. No Protocol Admin wrapper exists for overriding a pool admin’s base fee.

**Authority:** Non-renounced pool_admin.

**Arguments in order:** `swap_fee_rate: u32`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `SwapFeeRateUpdated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_swap_fee_rate.rs).

### cubic_pool.set_swaps_enabled

Discriminator: `909acd3af1a94028`.

Sets the swap gate; proportional liquidity operations are not gated by this flag.

**Authority:** Nonzero pool_admin OR config.protocol_admin.

**Arguments in order:** `enabled: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `SwapsEnabledUpdated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_swaps_enabled.rs).

### cubic_pool.set_token_active

Discriminator: `009eca23328bd912`.

For valid token_index, sets the input-only swap gate. False still permits output purchases and proportional liquidity operations.

**Authority:** Nonzero pool_admin OR config.protocol_admin.

**Arguments in order:** `token_index: u8`, `is_active: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `TokenActiveSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/set_token_active.rs).

### cubic_pool.swap

Discriminator: `f8c69e91e17587c8`.

Exact gross input; positive amount and different valid indices. Checks pool/trading flags and input is_active. Gross input advances selloff, base fee reduces priced input, dynamic fee reduces output. minimum_amount_out is net output floor. Output is bounded by LP actual reserve; transfers validate mints/programs/accounts.

**Authority:** User, or a CPI-signing user PDA.

**Arguments in order:** `amount_in: u64`, `minimum_amount_out: u64`, `token_in_index: u8`, `token_out_index: u8`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `token_mint_in` |  | — |
| `token_mint_out` |  | — |
| `user_token_account_in` | w | — |
| `user_token_account_out` | w | — |
| `vault_in` | w | — |
| `vault_out` | w | — |
| `user` | s | — |
| `token_program_in` |  | — |
| `token_program_out` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `MaxSelloffWindowAdvanced`, `Swap`, `PoolStateLog`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/swap.rs).

### cubic_pool.withdraw_sol

Discriminator: `91834a8841892a26`.

Transfers positive lamports above the rent floor from the supplied config itself or a CubicPool belonging to it. Source must be cubic-pool owned. Does not transfer SPL-token balances.

**Authority:** Nonzero config.protocol_admin.

**Arguments in order:** `amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `source` | w | — |
| `recipient` | w | — |
| `authority` | s | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolSolWithdrawn`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/withdraw_sol.rs).

## protocol_admin

### protocol_admin.accept_admin_transfer

Discriminator: `59d360d4e900fb07`.

Changes treasury.admin to the pending signer and clears pending_admin.

**Authority:** Nonzero treasury.pending_admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `new_admin` | s | — |

**Remaining accounts:** None required by this instruction.

**Event:** `AdminTransferred`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/admin_transfer.rs).

### protocol_admin.cancel_admin_transfer

Discriminator: `26839d1ff0892cd7`.

Clears pending_admin without changing the current Treasury admin.

**Authority:** Current Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `admin` | s | — |

**Remaining accounts:** None required by this instruction.

**Event:** `AdminTransferCancelled`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/admin_transfer.rs).

### protocol_admin.close_pool_program

Discriminator: `0e3d89dda939b6fd`.

Closes program through upgradeable loader and refunds ProgramData rent to recipient. A destructive lifecycle operation, not a reversible freeze.

**Authority:** Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `programdata` | w | — |
| `program_to_close` | w | — |
| `recipient` | w | — |
| `bpf_loader_upgradeable` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ProgramClosed`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/close_pool_program.rs).

### protocol_admin.freeze_pool_program

Discriminator: `fba0e2b2bc8690a9`.

Sets loader upgrade authority to None permanently. Program remains callable; this is not a pool operating pause.

**Authority:** Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `program` |  | — |
| `programdata` | w | — |
| `bpf_loader_upgradeable` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ProgramFrozen`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/freeze_pool_program.rs).

### protocol_admin.freeze_pools

Discriminator: `9142add2a2c8f816`.

Batch sets pool_enabled=false by Treasury CPIs for each remaining config/pool pair. Nonempty even-length layout; any failure reverts the whole transaction.

**Authority:** Treasury admin OR configured supervisor.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `authority` | s | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** Nonempty sequence of `[config_i, pool_i (w)]` pairs. Every config must authorize Treasury; every pool must belong to its paired config.

**Event:** `PoolsFrozen`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_freeze.rs).

### protocol_admin.initialize

Discriminator: `afaf6d1f0d989bed`.

One-time Treasury creation. Canonical ProgramData must identify payer as this program’s upgrade authority. admin argument must be nonzero; it may differ from payer.

**Authority:** Current upgrade authority as payer.

**Arguments in order:** `admin: Pubkey`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `payer` | w, s | — |
| `program_data` |  | — |
| `system_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `TreasuryInitialized`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/initialize.rs).

### protocol_admin.initiate_admin_transfer

Discriminator: `e8848574b8c51143`.

Writes pending_admin. This handler does not reject zero or self nomination; zero cannot accept. Use a deliberate nonzero successor and verify it before acceptance.

**Authority:** Current Treasury admin.

**Arguments in order:** `new_admin: Pubkey`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `admin` | s | — |

**Remaining accounts:** None required by this instruction.

**Event:** `AdminTransferInitiated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/admin_transfer.rs).

### protocol_admin.pool_accept_protocol_admin_transfer

Discriminator: `22a7c7740ecbd043`.

Accepts config authority specifically into Treasury PDA. Pending config authority must be Treasury; the human admin is not the inner accepting signer.

**Authority:** Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Event:** `PoolProtocolAdminTransferAccepted`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_protocol_admin_transfer.rs).

### protocol_admin.pool_cancel_protocol_admin_transfer

Discriminator: `b996a4faee491a8d`.

Cancels a pending config-authority transfer by Treasury-signed CPI.

**Authority:** Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Event:** `PoolProtocolAdminTransferCancelled`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_protocol_admin_transfer.rs).

### protocol_admin.pool_collect_protocol_fees

Discriminator: `84ba795c3c3dd8a8`.

Forwards to cubic_pool.collect_protocol_fees with Treasury as protocol authority; passes ordered recipients unchanged. Emits outer event plus the inner detailed collection event.

**Authority:** Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` |  | — |
| `pool` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** Exactly 3N: `[vault_i (w), recipient_i (w), token_program_i]` for each token in pool order.

**Events:** `PoolFeesCollected`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_collect_protocol_fees.rs).

### protocol_admin.pool_debug_withdraw_liquidity

Discriminator: `e948ed70b7b1d8d6`.

Forwards to disabled-pool emergency recovery; carries the same retirement/bookkeeping limitation as cubic_pool.debug_withdraw_liquidity.

**Authority:** Treasury admin.

**Arguments in order:** `token_amounts: Vec<u64>`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` |  | — |
| `pool` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** Exactly 3N: `[vault_i (w), recipient_i (w), token_program_i]` for each token in pool order.

**Events:** `PoolDebugWithdrawn`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_debug_withdraw_liquidity.rs).

### protocol_admin.pool_initialize_alt

Discriminator: `fc2d351f0a325268`.

Treasury authorizes the ALT CPI and human admin pays. Inner handler still requires non-renounced pool_admin. ALT address uses Treasury and recent_slot.

**Authority:** Treasury admin.

**Arguments in order:** `recent_slot: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | w, s | — |
| `pool` | w | — |
| `config` |  | — |
| `lookup_table` | w | — |
| `system_program` |  | — |
| `alt_program` |  | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolAltInitializedByTreasury`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_initialize_alt.rs).

### protocol_admin.pool_initialize_config

Discriminator: `d06eadbc279990b9`.

Treasury signs cubic_pool.initialize_config by CPI. Admin pays new-config rent; config is an explicit new signing account.

**Authority:** Treasury admin; config keypair also signs.

**Arguments in order:** `default_protocol_fee_rate: u16`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | w, s | — |
| `config` | w, s | — |
| `cubic_pool_program` |  | — |
| `system_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolConfigInitialized`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_initialize_config.rs).

### protocol_admin.pool_initiate_protocol_admin_transfer

Discriminator: `890a26d714e8cf87`.

Nominates new config protocol authority by Treasury-signed CPI; current config authority must be Treasury.

**Authority:** Treasury admin.

**Arguments in order:** `new_admin: Pubkey`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Event:** `PoolProtocolAdminTransferInitiated`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_protocol_admin_transfer.rs).

### protocol_admin.pool_migrate_to_v5

Discriminator: `99e1b28a081b9326`.

Treasury-signed migration; passes reactivate_tokens verbatim. Emits the inner PoolMigratedToV5 event; no additional wrapper event schema.

**Authority:** Treasury admin.

**Arguments in order:** `reactivate_tokens: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `admin` | w, s | — |
| `pool` | w | — |
| `config` |  | — |
| `system_program` |  | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_migrate_to_v5.rs).

### protocol_admin.pool_set_banned_extensions

Discriminator: `c0e37199435a30f5`.

Treasury-signed config update carrying both default and hard masks. Inner BannedExtensionsUpdated contains both masks; outer event omits hard-mask fields.

**Authority:** Treasury admin.

**Arguments in order:** `banned_extensions: u64`, `hard_banned_extensions: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolBannedExtensionsSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_set_banned_extensions.rs).

### protocol_admin.pool_set_pool_enabled

Discriminator: `2e4629b4822b5d8c`.

Treasury-signed CPI to set_pool_enabled. Supervisor uses freeze_pools/unfreeze_pools instead.

**Authority:** Treasury admin.

**Arguments in order:** `enabled: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` |  | — |
| `pool` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolEnabledSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_set_pool_enabled.rs).

### protocol_admin.pool_set_protocol_fee_rate

Discriminator: `85301c8a63556f4d`.

Treasury-signed CPI to set_protocol_fee_rate; config must belong to this pool and authorize Treasury.

**Authority:** Treasury admin.

**Arguments in order:** `protocol_fee_rate: u16`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` |  | — |
| `pool` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolProtocolFeeRateSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_set_protocol_fee_rate.rs).

### protocol_admin.pool_set_swaps_enabled

Discriminator: `2ba6b95e23773a01`.

Treasury-signed CPI to set_swaps_enabled. Does not change pool_enabled.

**Authority:** Treasury admin.

**Arguments in order:** `enabled: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` |  | — |
| `pool` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolSwapsEnabledSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_set_swaps_enabled.rs).

### protocol_admin.pool_set_token_active

Discriminator: `d2e15b0d24b155f2`.

Treasury-signed per-token input toggle. Both deactivation and reactivation are allowed; not freeze-only.

**Authority:** Treasury admin OR configured supervisor.

**Arguments in order:** `token_index: u8`, `is_active: bool`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `authority` | s | — |
| `config` |  | — |
| `pool` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolTokenActiveSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_set_token_active.rs).

### protocol_admin.pool_withdraw_sol

Discriminator: `1d304e6585b34ee6`.

Treasury-signed scoped cubic-pool SOL recovery; source must be the supplied config or its pool and rent is preserved.

**Authority:** Treasury admin.

**Arguments in order:** `amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` |  | — |
| `source` | w | — |
| `recipient` | w | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `PoolSolWithdrawnViaTreasury`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_withdraw_sol.rs).

### protocol_admin.register_token

Discriminator: `209224f050b72454`.

Creates classic SPL Token vault PDA ["vault", mint], owned by Treasury, and appends mint/vault to up to ten registered entries. Admin pays rent. Token-2022 is not accepted by these typed accounts.

**Authority:** Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `mint` |  | — |
| `vault` | w | — |
| `admin` | w, s | — |
| `token_program` |  | — |
| `system_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `TokenRegistered`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/register_token.rs).

### protocol_admin.set_supervisor

Discriminator: `2dcdc6666e843726`.

Writes or revokes supervisor; zero removes the role. Reallocates an old 786-byte Treasury to 818 bytes when needed, admin pays rent difference.

**Authority:** Treasury admin.

**Arguments in order:** `new_supervisor: Pubkey`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `admin` | w, s | — |
| `system_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `SupervisorSet`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/set_supervisor.rs).

### protocol_admin.stld_withdraw_sol

Discriminator: `c891211e7e6ad685`.

Treasury-signed STLD recovery from the specified pool helper PDA. Pool must belong to config and config must authorize Treasury; helper is dataless, so full recovery is allowed.

**Authority:** Treasury admin.

**Arguments in order:** `amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `config` |  | — |
| `pool` |  | — |
| `source` | w | — |
| `recipient` | w | — |
| `system_program` |  | — |
| `stld_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `StldSolWithdrawnViaTreasury`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/stld_withdraw_sol.rs).

### protocol_admin.transfer_upgrade_authority

Discriminator: `5234753538c4fddb`.

Uses SetAuthorityChecked with Treasury as current authority; canonical ProgramData is derived from program. Incoming signer becomes upgrade authority.

**Authority:** Treasury admin AND new_authority signer.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `program` |  | — |
| `programdata` | w | — |
| `new_authority` | s | — |
| `bpf_loader_upgradeable` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `UpgradeAuthorityTransferred`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/transfer_upgrade_authority.rs).

### protocol_admin.unfreeze_pools

Discriminator: `13cd2280cde46970`.

Same batch authorization/layout as freeze_pools; sets pool_enabled=true. Supervisor is permitted to unfreeze in this branch.

**Authority:** Treasury admin OR configured supervisor.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `authority` | s | — |
| `cubic_pool_program` |  | — |

**Remaining accounts:** Nonempty sequence of `[config_i, pool_i (w)]` pairs, with the same checks as freeze_pools.

**Event:** `PoolsUnfrozen`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/pool_freeze.rs).

### protocol_admin.upgrade_pool_program

Discriminator: `26380640eee10308`.

Treasury invokes upgradeable loader. Treasury must own upgrade authority and buffer authority; program and ProgramData must match loader expectations. spill receives remaining buffer rent. Program target can include Protocol Admin itself.

**Authority:** Treasury admin.

**Arguments in order:** None.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `admin` | s | — |
| `programdata` | w | — |
| `program_to_upgrade` | w | — |
| `buffer` | w | — |
| `spill` | w | — |
| `rent` |  | — |
| `clock` |  | — |
| `bpf_loader_upgradeable` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `ProgramUpgraded`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/upgrade_pool_program.rs).

### protocol_admin.withdraw

Discriminator: `b712469c946da122`.

Transfers raw classic SPL tokens from a Treasury-owned token account to a recipient. The context checks Treasury ownership; it does not require the source to occur in the registration array. This is not a Token-2022 TransferChecked wrapper.

**Authority:** Treasury admin.

**Arguments in order:** `amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` |  | — |
| `vault` | w | — |
| `recipient` | w | — |
| `admin` | s | — |
| `token_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `FundsWithdrawn`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/withdraw.rs).

### protocol_admin.withdraw_sol

Discriminator: `91834a8841892a26`.

Transfers a positive amount of Treasury lamports while preserving its rent-exempt minimum.

**Authority:** Treasury admin.

**Arguments in order:** `amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `treasury` | w | — |
| `admin` | s | — |
| `recipient` | w | — |

**Remaining accounts:** None required by this instruction.

**Events:** `SolWithdrawn`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/protocol-admin/src/instructions/withdraw_sol.rs).

## single_token_liquidity

### single_token_liquidity.deposit_single_token

Discriminator: `a688a62fc7c056a9`.

Atomic zap for 2–10 token seeded pools: input actual reserve > 0, amount_in > 0 and minimum_bpt_amount > 0. Splits input using deployed allocation math, executes swaps in token order, reloads pool, caps helper basket, joins, transfers only newly minted BPT and refunds all token dust. Internal swaps use min_out=0; final BPT minimum guards the whole sequence. Uses existing helper token balances too.

**Authority:** User.

**Arguments in order:** `amount_in: u64`, `token_in_index: u8`, `minimum_bpt_amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `pool` | w | — |
| `helper` |  | — |
| `bpt_mint` | w | — |
| `helper_bpt_account` | w | — |
| `user_bpt_account` | w | — |
| `user` | w, s | — |
| `cubic_pool_program` |  | — |
| `bpt_token_program` |  | — |

**Remaining accounts:** At least 5N accounts, first 5N as `[mint_i, user_token_i (w), helper_ata_i (w), vault_i (w), token_program_i]` per token. Supply the exact 5N layout in clients. Canonical helper/pool ATAs; user token-account mint/owner are checked. All required token/BPT accounts must be prepared before the instruction.

**Events:** `SingleTokenDeposit`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/single-token-liquidity/src/instructions/user/deposit_single_token.rs).

### single_token_liquidity.withdraw_sol

Discriminator: `91834a8841892a26`.

Helper PDA ["stld_helper", pool] signs a System transfer. Requires pool.config match, positive amount ≤ helper balance. Dataless System-owned helper has no rent floor to preserve.

**Authority:** Nonzero config.protocol_admin.

**Arguments in order:** `amount: u64`.

| Fixed account in order | Flags | IDL-pinned address |
| --- | --- | --- |
| `config` |  | — |
| `pool` |  | — |
| `source` | w | — |
| `recipient` | w | — |
| `authority` | s | — |
| `system_program` |  | — |

**Remaining accounts:** None required by this instruction.

**Events:** `StldSolWithdrawn`.

[Handler source](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/single-token-liquidity/src/instructions/admin/withdraw_sol.rs).
