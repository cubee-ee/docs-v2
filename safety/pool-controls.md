# Pool Controls

This page describes the executable authority checks in contracts `audit-fixes-excluded-SF` at [`96a2ee2`](https://github.com/coffer-so/contracts/tree/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs), as exposed by SDK 0.11.1. It does not infer governance structure from a PDA address or assume the frontend exposes every instruction.

## Authorities

| Role | Stored on | Powers |
| --- | --- | --- |
| Pool admin | `pool.pool_admin`; initially the creator/payer | Base swap fee, token input and swap toggles, selloff/surge policy, range-manager appointment and limits, pool-admin succession and renunciation, first seed deposit, ALT |
| Config protocol authority | `config.protocol_admin`; initially Treasury PDA | Pool operating switch, protocol fee share and collection, extension defaults/floor, token input and swap toggles, emergency withdrawal, scoped SOL recovery, config-authority succession, migration, ALT |
| Range manager | `pool.range_manager` plus enabled flag | `range_manager_update` under its configured bounds |
| Treasury admin | `treasury.admin` | All Protocol Admin wrappers, Treasury withdrawals, program lifecycle, supervisor assignment and Treasury succession |
| Treasury supervisor | `treasury.supervisor`; zero means absent | Batch pool freeze **and unfreeze**, token input disable **and enable** |

The pool creator usually signs their own direct Cubic Pool instructions. The config's protocol authority may be a wallet or PDA after a valid transfer; when it is the Treasury PDA, use the corresponding `protocol_admin` wrapper. Only that program can sign for its Treasury PDA. There is no automatic multisig or timelock in this code; wallet or governance security must be evaluated separately.

Authority checks still apply when a pool is paused. A full pause blocks trading, LP operations, and range-manager updates; it does not disable recovery and administrative controls needed to manage the pause.

## Operating switches

| Control | Signer path | Effect |
| --- | --- | --- |
| `set_pool_enabled(false)` | Config protocol authority; Treasury admin through `pool_set_pool_enabled` | Stops swaps, LP add/remove, STLD and range-manager updates |
| `freeze_pools` / `unfreeze_pools` | Treasury admin or configured supervisor | Sets `pool_enabled` false/true for each `[config, pool]` pair, atomically within one transaction |
| `set_swaps_enabled(false)` | Pool admin or config protocol authority; Treasury admin wrapper | Stops direct swaps and STLD, leaves proportional LP operations available |
| `set_token_active(index, false)` | Pool admin or config protocol authority; Treasury admin/supervisor wrapper | Rejects the token only as swap input; buying it as output remains possible |

The supervisor in this branch is not freeze-only. It can reverse the pauses and token disables it is allowed to make. Revoking it with `set_supervisor(Pubkey::default())` removes that role; it does not reverse any existing pool or token flags. The first supervisor assignment to an old Treasury account can grow it from 786 to 818 bytes, paid by the admin.

An inactive token and a sidelined token are different. Inactive means `is_active=false`; sidelined means `actual_balance=0`. A sidelined token cannot be paid out until funded, but if input-active it may become live through a swap into the pool. Proportional deposits cannot independently revive a zero-reserve slot.

## Fees and selloff configuration

Only the pool admin sets `swap_fee_rate`, bounded to 100,000 on a 1,000,000 scale (10%). Config protocol authority sets `protocol_fee_rate`, bounded to 5,000 on a 10,000 scale (50% of the base swap fee). There is no Treasury wrapper to replace a pool admin's base-fee setting.

`set_max_selloff(params)` is pool-admin only. It replaces a complete vector of `SelloffParams`, one per token, with the cap, window period, dynamic-fee threshold, low/mid/high rates and optional kink. It does not reset the accumulated window state. A cap reduction can therefore reject the next swap immediately. `max_selloff_pct=0` disables both the limiter and surge for that token; zero high rate disables surge while leaving an enabled limiter.

The cap is a percentage of a virtual-balance snapshot captured for the window, not a fixed number of tokens and not a percentage of actual reserves. Gross swap input consumes headroom. The fee is based on input-token window utilization but charged in the output token, entirely into protocol fees. See the full parameter table in [pool parameters](../overview/pool-parameters.md#selloff-and-dynamic-fee-policy), the [window algorithm](../for-traders/max-selloff.md), and [math](../technical/math.md).

## Range manager

`set_range_manager(new_manager, enabled)` gives the pool admin full appointment and enable/disable control. A nonzero manager must be enabled to operate; a zero key cannot operate even if the flag is true. `set_range_manager_config(max_vb_change_pct, max_weight_change_pct, min_update_interval_secs, max_leverage_bps, min_leverage_bps)` is pool-admin only.

The Cubic Pool handler additionally permits the config protocol authority to disable the existing manager without changing its pubkey. However, the current Protocol Admin program has no `pool_set_range_manager` CPI wrapper. When the config authority is the Treasury PDA, this specific direct disable path is not reachable through the deployed wrappers; Treasury's full pool pause remains available.

A manager submits sparse `vb_changes` and `weight_changes`. Each entry is:

```text
TokenChange { index: u8, expected_current: u64, new_value: u64 }
```

`expected_current` is mandatory: the stored value must still match the manager's observed value, or the update fails with `RangeManagerStaleValue`. This prevents a stale quote from overwriting intervening changes to the fields being written. It is a per-field check, not a lock on the entire pool snapshot.

Every successful update must satisfy:

1. Pool enabled, manager enabled and nonzero, signer equals manager.
2. Minimum interval elapsed since the last successful update.
3. At least one change; valid token indices, no duplicates within a list, positive new values, and matching expected values.
4. For each changed field, `abs(new − old) × 10,000 ≤ old × configured_change_pct`. Both caps are at most 10,000. A zero cap permits no nonzero change.
5. Final weights each remain 100–9,900 and sum to 10,000.
6. For virtual-balance slots written by this call with nonzero actual reserves, the new virtual/actual ratio lies within every enabled minimum/maximum leverage bound. Each bound uses 10,000 = 1×; zero disables that side.

Both ratio bounds may exceed 10,000, but if both are set, minimum must not exceed maximum. The absolute band is checked only for virtual-balance entries written by the call; a weight-only change does not write a virtual balance. Zero-reserve slots are exempt because the ratio is undefined. This allows correction of one slot when another drifted out of band through swaps or LP operations.

The manager changes virtual balances and weights without transferring tokens. These changes can change prices and affect LP value. Percentage caps bound one step; repeated steps can compound. Time delay and the two absolute ratio bounds are separate controls. No manager update is an external oracle-price check.

## Admin succession and renunciation

There are three independent two-step transfers:

| Authority being changed | Nominate/cancel | Accept |
| --- | --- | --- |
| `pool.pool_admin` | Current pool admin | `pool.pending_pool_admin` signs `accept_pool_admin_transfer` |
| `config.protocol_admin` | Current config protocol authority | `config.pending_protocol_admin` signs `accept_protocol_admin_transfer`; Treasury acceptance uses its wrapper |
| `treasury.admin` | Current Treasury admin | `treasury.pending_admin` signs `accept_admin_transfer` |

The old admin retains power until acceptance. Cancelling clears only the pending successor. Pool/config nomination rejects zero. Treasury nomination writes the supplied key without a zero/self check, but zero cannot accept; use a deliberate nonzero successor. Config-authority rotation does not change Treasury admin or a program's upgrade authority. After rotation away from Treasury, its protocol-authority-only wrappers and supervisor circuit breaker no longer authorize those pools; the independent pool-admin paths follow their own checks. Treasury-admin rotation changes who may request Treasury-signed actions without changing the Treasury PDA itself.

`disable_pool_admin` permanently zeros both current and pending pool-admin fields. It removes pool-admin-only configuration access and prevents an unseeded pool's first deposit. It **does not** clear or disable an existing range manager and does not remove config/Treasury powers. It therefore does not make every economic parameter immutable. If the intended final state has no range-manager authority, disable the manager before renouncing the pool admin.

`initialize_pool_alt` also requires a non-renounced pool admin even on the protocol-authority path. Provision an ALT before renunciation if needed. Creation, extension with pool-scoped addresses, and removal of the ALT authority occur in one instruction; a second initialization is rejected. The separate payer funds the table, and the ALT's address uses the signing authority and supplied recent slot. Existing ALT addresses are stored on the pool.

## Token extension policy

Token admission checks run during pool creation. The stored pool bitmap documents the effective policy at that point; changing the config's bitmaps does not revalidate or rewrite existing pools.

```text
hard_floor = config.hard_banned_extensions == 0 ? 512 : config.hard_banned_extensions
effective_bitmap = (override if supplied, else config.banned_extensions) OR hard_floor
```

New configs initialize the default bitmap to **100684810**, covering TransferFeeConfig (1), MintCloseAuthority (3), InterestBearingConfig (10), PermanentDelegate (12), TransferHook (14), ScaledUiAmount (25), and Pausable (26). New configs initialize the hard floor to **512**, NonTransferable (9). These are code defaults; inspect stored values for an existing config.

Independent of those bitmaps, creation rejects NonTransferable, Token-2022 native mint, malformed/non-mint or uninitialized accounts, unknown extension types above 27, a `DefaultAccountState` other than Initialized, and a TransferHook whose program ID is nonzero. Clearing a bitmap cannot override these checks. Classic mint size is 82 bytes; an extended Token-2022 mint must have its mint account-type tag and a valid TLV layout.

The policy can admit issuer powers when a creator deliberately clears default bits and the hard floor permits it. A PermanentDelegate can affect vault custody; an issuer can freeze or pause transfers, or arm an initially inactive transfer hook later. Raw AMM balances do not automatically incorporate UI amount scaling or interest display rules. Admission is not an assurance that issuer state remains unchanged.

Runtime transfer compatibility is a separate restriction in this without-SF deployment. Its liquidity/helper/fee-recovery paths use plain token `Transfer` CPIs and do not implement transfer-fee or transfer-hook account handling. SDK 0.11.1 rejects TransferFeeConfig, TransferHook, NonTransferable, ConfidentialTransferFeeConfig, Pausable and unknown extension types in normal builders for affected tokens; the STLD builder conservatively checks every pool mint because helper dust may also be transferred. Mint parsing rejects malformed data. These guards do not replace checking current token-account state or simulating the intended transaction. A stored ban bit by itself is a disclosure flag rather than proof that an existing transfer must fail. Use the SDK's runtime diagnostics and current mint data; do not treat permission to initialize a pool as permission to execute every operation.

## Protocol fee collection and recovery

`collect_protocol_fees` transfers the owed amount from each canonical pool vault into a recipient account of the same mint and token program, then clears `protocol_fees_owed`. The recipient wallet is selected by the authorized caller; it is not forced to the Treasury. Zero-fee slots are skipped. LP actual and virtual balances stay unchanged.

`debug_withdraw_liquidity` is an emergency recovery operation on an already disabled pool. It transfers specified token amounts but does not reconcile actual/virtual balances or BPT supply. An affected pool must remain retired: the handler does not write an irreversible retirement flag, so governance must prevent later re-enabling it. It is not an ordinary LP exit or fee collection.

SOL recovery is scoped differently by program. Cubic Pool permits excess lamports above rent only from the authorizing config account or a pool belonging to that config. Treasury's own SOL withdrawal preserves its rent floor. STLD recovery is limited to the helper PDA of a pool belonging to the supplied config; this helper is System-owned and dataless, so its balance can be recovered in full. These paths do not withdraw pool SPL-token liquidity.

Treasury `register_token` and `withdraw` support classic SPL Token accounts in this contract; they are not general Token-2022 wrappers. Registration creates up to ten Treasury vault PDAs. Withdrawal authorizes a Treasury-owned classic token account; token-program checks enforce matching token semantics.

## Program lifecycle

Treasury admin can invoke `upgrade_pool_program` for a program whose upgrade authority is Treasury. The loader verifies the program, ProgramData and buffer. The buffer must be prepared with the correct authority; remaining buffer rent goes to the specified spill account. This operation can upgrade Cubic Pool, STLD or Protocol Admin itself when authority matches.

`transfer_upgrade_authority` uses the checked loader operation: the incoming authority co-signs, and becomes the new upgrade authority. `freeze_pool_program` removes upgrade authority permanently while leaving program execution available. `close_pool_program` closes the program through the loader and refunds ProgramData rent to the recipient; it is a destructive program lifecycle operation, not a reversible pool pause.

See [instruction reference](../technical/instruction-reference.md) for exact argument order, accounts, signers and CPI wrappers, and [accounts/events](../technical/accounts-events.md) for monitoring fields and known event omissions.
