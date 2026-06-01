# Pool Controls

Cube pools expose a layered authority model: **per-pool admin** (`pool_admin`), **protocol-wide admin** (`protocol_admin` → Treasury PDA), and an optional **range manager** (delegated rebalancer). Each authority has a specific subset of instructions it can call. This page enumerates every admin instruction on the `cubic_pool` program, what it does, who can sign it, and how the on-chain handler validates inputs.

The frontend exposes most of these as forms on the **Admin Panel** (`/pool-admin/:poolAddress`) and **Range Manager Panel** (`/pool-manager/:poolAddress`). The on-chain source of truth is `programs/cubic-pool/src/instructions/`.

---

## Authority model

| Authority | Pubkey field | Set how | Owns |
|---|---|---|---|
| `pool_admin` | `pool.pool_admin` | Set at `initialize_cubic_pool`, transferable 2-step | Swap fee, swap toggle, range manager (set + config), max-selloff, ALT init, admin transfer / renounce |
| `protocol_admin` | `config.protocol_admin` (resolves to Treasury PDA via protocol-admin program) | Set at config init | Master `pool_enabled` kill, banned-extension bitmap, protocol-fee rate, protocol-fee collection. Also has CPI wrappers for some pool-admin ops. |
| `range_manager` | `pool.range_manager` | Set by `pool_admin` via `set_range_manager` | `range_manager_update` only (within configured envelope) |

Both admin pubkeys are pubkeys, not necessarily multisigs — but in production both resolve to PDAs (Treasury, governance) so no single key can act unilaterally.

---

## Pool-admin instructions

All require `pool.pool_admin` to sign. Most are exposed as one-form-each on the **Admin Panel**.

### `set_swap_fee_rate`

```rust
set_swap_fee_rate(ctx: Context<SetSwapFeeRate>, swap_fee_rate: u32)
```

Update the swap fee. Value is in **hundredths of a basis point** (`SWAP_FEE_PRECISION = 1_000_000`): `30_000` = 3%, `100_000` = 10% (current cap).

Constraints: `swap_fee_rate ≤ MAX_SWAP_FEE_RATE` (= 100_000 → 10%). Reverts with `FeeRateMaxExceeded` otherwise.

Frontend: **Admin Panel → Set swap fee**. Input is in **%**; the page converts to hundredths of bps (`pct × 10_000`).

Effect: every subsequent swap deducts this fee from `amount_in`. LP-share = `swap_fee × (1 − protocol_fee_rate)`. See [Swapping → Fee Mechanics](../for-traders/swapping.md#fee-mechanics).

### `set_swaps_enabled`

```rust
set_swaps_enabled(ctx: Context<SetSwapsEnabled>, enabled: bool)
```

Per-pool swap toggle. Less invasive than `set_pool_enabled` — LPs can still `add_liquidity` / `remove_liquidity` while swaps are paused.

Authority: `pool_admin` OR `protocol_admin` (the latter via a CPI from the protocol-admin program). Both work.

Frontend: **Admin Panel → Enable / disable swaps**.

Use case: pause trading during an investigation, while letting LPs exit cleanly.

### `set_range_manager`

```rust
set_range_manager(
    ctx: Context<SetRangeManager>,
    new_manager: Pubkey,
    enabled: bool,
)
```

Set or rotate the range-manager pubkey + the enabled gate. The range manager is the only authority that can call `range_manager_update`.

Pass `new_manager = Pubkey::default()` with `enabled = false` for the fully-disabled state. New pools deploy with this off by default.

Frontend: **Admin Panel → Set range manager**. Two fields: pubkey + enabled checkbox.

### `set_range_manager_config`

```rust
set_range_manager_config(
    ctx: Context<SetRangeManagerConfig>,
    max_vb_change_bps: u16,
    max_weight_change_bps: u16,
    min_update_interval_secs: u32,
)
```

Configure the envelope the range manager must stay within on every update:

| Param | Range | Meaning |
|---|---|---|
| `max_vb_change_bps` | 0–10 000 | Per-update change in `virtual_balance[i]` (bps of current value). `500` = ±5%. |
| `max_weight_change_bps` | 0–10 000 | Per-update change in `normalized_weight[i]`. Same units. |
| `min_update_interval_secs` | `u32` | Minimum seconds between consecutive `range_manager_update` calls. Rate-limits how fast the manager can move the curve. |

Constraints: both bps values must be ≤ 10 000 (= ±100%), otherwise reverts with `RangeManagerInvalidBps`.

Frontend: **Admin Panel → Set range manager config**.

Effect: limits how aggressively the range manager can drift the curve. Without this envelope a misconfigured manager could move spot prices arbitrarily far in one tx.

### `set_max_selloff`

```rust
set_max_selloff(
    ctx: Context<SetMaxSelloff>,
    max_selloffs: Vec<u64>,    // length = token_count, raw units
    period_lengths: Vec<u32>,  // length = token_count, seconds
)
```

Set per-token sliding-window sell-side rate limits. Both vectors must be **exactly `token_count` long** — sparse updates aren't supported; to leave a token unchanged, repeat its current value.

`max_selloff = 0` disables the check for that slot (period is ignored).

Frontend: **Admin Panel → Set max-selloff window (per-token)**. Renders one card per token with current values pre-filled, accepts inputs in **human units** (e.g. "10" BONK), and converts to raw `u64` via `10^decimals` on send.

Effect: blocks swaps that would push that token's rolling sell volume past the cap. See [Max-Selloff Window](../for-traders/max-selloff.md) for the math.

### `initiate_pool_admin_transfer`

```rust
initiate_pool_admin_transfer(
    ctx: Context<InitiatePoolAdminTransfer>,
    new_admin: Pubkey,
)
```

Step 1 of a 2-step admin transfer. Writes `new_admin` to `pool.pending_pool_admin`. The current `pool_admin` retains all rights until the new admin claims.

Frontend: **Admin Panel → Initiate admin transfer**.

### `cancel_pool_admin_transfer`

```rust
cancel_pool_admin_transfer(ctx: Context<CancelPoolAdminTransfer>)
```

Wipe `pending_pool_admin` back to `Pubkey::default()` — cancels a pending handover. Signed by the current `pool_admin`.

Frontend: **Admin Panel → Cancel admin transfer**.

### `accept_pool_admin_transfer`

```rust
accept_pool_admin_transfer(ctx: Context<AcceptPoolAdminTransfer>)
```

Step 2 of the transfer. **Must be signed by the wallet stored in `pool.pending_pool_admin`** — not the outgoing admin. On success: `pool_admin := pending_pool_admin`, `pending_pool_admin := default`.

Frontend: **Admin Panel → Accept admin transfer**. The UI gates the submit button until the connected wallet matches `pending_pool_admin` and instructs the user to switch wallets in their extension if needed.

### `disable_pool_admin`

```rust
disable_pool_admin(ctx: Context<DisablePoolAdmin>)
```

Renounce the pool-admin role. Sets `pool.pool_admin = Pubkey::default()`. **Irreversible** — once disabled, no swap-fee changes, no range-manager updates, no further admin transfers are possible. Use only when the pool's parameters are finalised and you want to credibly commit to immutability.

Frontend: not currently exposed (too dangerous for a single click). Available via SDK / CLI.

### `withdraw_sol`

```rust
withdraw_sol(ctx: Context<WithdrawSol>, amount: u64)
```

Pool-admin recovers any stray native SOL deposited to the pool PDA. Does NOT touch SPL token vaults — those are governed by `add_liquidity` / `remove_liquidity` only.

### `initialize_pool_alt`

```rust
initialize_pool_alt(ctx: Context<InitializePoolAlt>, recent_slot: u64)
```

Provision a per-pool Address Lookup Table containing pool PDA, BPT mint, all vault ATAs, all mints, and standard programs. Address is stored in `pool.lookup_table`. The ALT is **frozen** in the same handler — no later mutation possible, so depositors don't have to trust the admin not to rug accounts.

Used by the create-pool flow on the frontend to enable VersionedTransactions for 7+ token pools. See `MULTI_TOKEN_POOL_ALT_DESIGN.md` in the contracts repo for the full rationale.

### `migrate_pool_v4`

```rust
migrate_pool_v4(ctx: Context<MigratePoolV4>)
```

One-shot account-format migration from the v3 layout (1154 bytes, parallel arrays) to v4 (1683 bytes, AoS). Idempotent: re-running on an already-migrated pool is a no-op. Required exactly once per pre-existing pool after the cubic-pool program was upgraded to a binary built against the v4 struct.

New pools created via `initialize_cubic_pool` skip migration (born v4).

---

## Range-manager instructions

Signed by `pool.range_manager`. Only one ix in this category:

### `range_manager_update`

```rust
range_manager_update(
    ctx: Context<RangeManagerUpdate>,
    vb_changes: Vec<TokenChange>,
    weight_changes: Vec<TokenChange>,
)
```

Where `TokenChange = { index: u8, new_value: u64 }` — both arrays are **sparse**: only include entries for tokens you're actually changing.

Per-tx constraints (checked on chain):

1. `range_manager_enabled = true`.
2. Caller signature = `range_manager`.
3. At least `range_manager_min_update_interval_secs` have elapsed since `range_manager_last_updated`.
4. For each `vb_changes` entry: `|new_vb − old_vb| / old_vb ≤ max_vb_change_bps`.
5. For each `weight_changes` entry: `|new_w − old_w| / old_w ≤ max_weight_change_bps`.
6. Sum of resulting weights = 10 000 across all tokens.
7. Each weight in `[MIN_WEIGHT, MAX_WEIGHT]`.
8. No duplicate `index` within either vector.

Reverts (atomic — no state changes on rejection):

| Error | Trigger |
|---|---|
| `RangeManagerDisabled` | The `enabled` flag is false |
| `RangeManagerUnauthorized` | Caller != `range_manager` |
| `RangeManagerUpdateTooFrequent` | Min interval not yet elapsed |
| `RangeManagerVbChangeTooLarge` | One VB delta exceeds bps cap |
| `RangeManagerWeightChangeTooLarge` | One weight delta exceeds bps cap |
| `RangeManagerEmptyUpdate` | Both vectors empty (nothing to do) |
| `RangeManagerInvalidBps` | A bps value out of range |
| `RangeManagerWeightSumInvalid` | Final weights don't sum to 10000 |
| `RangeManagerDuplicateIndex` | Same index appears twice in one vector |

Frontend: **Range Manager Panel** has one form with per-token cards. Each card shows current weight + leverage + balances + price range, accepts new weight (%) and new leverage. Live preview: as the operator edits, the UI computes the projected new `virtual_balance` and the projected new price range against the projected base token state — so the manager can see exactly how the AMM spot price will shift before signing.

Effect: changes the curve without changing actual balances. Used to rebalance towards real market prices (so arb bots have less room to extract value) or to widen/tighten the curve in anticipation of expected flow.

---

## Protocol-admin instructions

Signed by `config.protocol_admin` (Treasury PDA). Not on the per-pool Admin Panel — exposed via separate operator tooling.

| Instruction | Effect |
|---|---|
| `set_pool_enabled(enabled)` | Master kill switch. `false` blocks **all** operations (swap, add, remove). Use for emergency or migration. |
| `set_protocol_fee_rate(rate_bps)` | Share of each swap fee taken by the protocol (bps; cap = 5000 = 50%). |
| `set_banned_extensions(banned: u64)` | Bitmap of Token-2022 extensions that are rejected at pool init. Tightening is safe; loosening means new pools can include previously banned extensions but existing pools are unaffected. |
| `collect_protocol_fees` | Sweep accumulated `protocol_fees_owed[i]` from each vault to the treasury ATA. Decreases `actual` and `virtual` proportionally so leverage stays intact. |
| `debug_withdraw_liquidity` | Emergency-only — drain a disabled pool to recover stranded funds. Requires `pool_enabled = false`. Pool stays retired afterwards. |

Most of these are also reachable through the protocol-admin program's CPI wrappers (with the Treasury PDA signing on behalf of the operator multisig).

---

## Reading the current authority state

Every admin field is on the `CubicPool` account and readable via:

- On-chain: `connection.getAccountInfo(pool)` + decode (see [Tracking Pool Activity → Direct Account Deserialization](../for-lps/tracking-pool-activity.md#direct-account-deserialization)).
- SDK: `await client.sync()` then `client.getCached()`. Fields are exposed on `PoolInfo`.
- Backend API: `GET /api/pools/:address` returns `poolAdmin`, `pendingPoolAdmin`, `rangeManager`, `rangeManagerEnabled`, the envelope fields, and `lookupTable`. See [API Reference](../integration/api-reference.md).

The frontend's Admin Panel shows the same fields in a header block with "= connected" plaques highlighting which role(s) the wallet you're connected with currently holds.

---

## Aggregator guidance

If you're routing swaps through Cube pools:

- Treat `pool_enabled = false` OR `swaps_enabled = false` as **inactive** — skip for routing.
- Treat any pool where the sliding-window selloff cap would block your `amount_in` as **rate-limited** — pre-compute headroom (see [Max-Selloff Window](../for-traders/max-selloff.md)) and either reduce `amount_in` for that leg or route around it.
- `range_manager_update` can shift spot prices between sync and execution. Use a fresh `sync()` (≤ 1 slot old) when quoting.
