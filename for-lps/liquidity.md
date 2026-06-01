# Liquidity (Add / Remove)

This page covers both sides of LP flow against a Cube pool — depositing and withdrawing.

The on-chain context for both is the **same Anchor account struct** (`ModifyLiquidity`), so most of the integration details (account layout, leverage maintenance, slippage protection) are shared. Differences are called out per section.

---

## Common context

### Pool state required

| Flag | Add | Remove |
|---|---|---|
| `pool_enabled = true` | required | required |
| `swaps_enabled` | not checked | not checked |

LPs can always add/remove while `pool_enabled`. The `swaps_enabled` toggle only blocks the `swap` instruction.

### Slippage protection

Every liquidity instruction takes a slippage floor:

- **Add:** `minimum_bpt_amount` — revert with `InsufficientBptOut` if the computed BPT mint would be smaller.
- **Remove:** `minimum_token_amounts: Vec<u64>` (one per token) — revert with `InsufficientTokensOut` if any per-token output would be smaller.

Both protect against front-running, large concurrent trades between quote and execution, and unexpected state changes.

### Leverage maintenance

Both directions update `virtual_balance` proportionally to actual flow so the **per-token leverage ratio (`virtual / actual`) stays constant** through normal LP activity:

```
delta_virtual_i = (amount_i / old_actual_i) × virtual_balance_i   // proportional scaling
```

Range-manager updates are the only way to change leverage outside of init.

### `remaining_accounts` layout

Both instructions take 4 trailing accounts per token in pool order, but **the ordering between (user, vault) and (vault, user) differs by direction** — it follows the source→destination flow.

Across both directions the layout is:

```
[<per-token-pair>] × N  +  [mint_0, mint_1, …]  +  [token_program_0, token_program_1, …]
```

| | Add | Remove |
|---|---|---|
| `i*2`     | `user_token_i` (source) | `vault_i` (source) |
| `i*2 + 1` | `vault_i` (destination) | `user_token_i` (destination) |
| `N*2 + i` | `mint_i` | `mint_i` |
| `N*3 + i` | `token_program_i` | `token_program_i` |

> **Note the swap.** Add: user→vault, so `user_token_i` comes first. Remove: vault→user, so `vault_i` comes first. The contract validates each direction independently.

### Vault validation

For each token, the contract derives the expected ATA at runtime
(`pool` × `mint` × `token_program`) and rejects any `vault_i` that
doesn't match. User token accounts are validated for owner + mint.
You can't be tricked into a malicious vault.

---

# Adding Liquidity

Cube supports two deposit modes:

1. **Proportional deposit** — you supply all tokens in the pool simultaneously, matching the current pool ratio. BPT is minted to you proportional to your contribution.
2. **Single-token deposit** — you supply just ONE of the pool's tokens; a helper program splits the input across every leg, swaps internally as needed, and mints BPT. **🚧 In development — not in production yet.** See [SDK / Single-token deposit](../sdk/single-token-deposit.md) for status.

The UI's main path is proportional mode. Power users can still call the on-chain `add_liquidity` directly.

## How proportional add works

### First deposit (empty pool)

When a pool has no liquidity (`bpt_total_supply == 0`):

1. User provides a positive `token_amounts[i]` for **every** pool token.
2. The invariant is computed from the pool's virtual balances using the weighted-product formula: `I = ∏ (balance_i ^ weight_i)`.
3. BPT minted equals the invariant value.
4. A minimum BPT threshold (`MINIMUM_INITIAL_BPT = 1_000` raw units) is enforced to prevent pool bricking.
5. Actual balances are set to the deposited amounts.

### Subsequent deposits

When a pool already has liquidity:

1. User provides `token_amounts` for each token (all must be > 0).
2. For each token, the ratio `amount / lp_accessible_balance` is computed, where `lp_accessible_balance = actual_balance − protocol_fees_owed`.
3. `bpt_minted = bpt_supply × min(ratio_i)` — the **minimum ratio** across all tokens determines the BPT amount.
4. Actual balances increase by the deposited amounts.
5. Virtual balances increase proportionally to maintain leverage.

The minimum-ratio rule means if you deposit more of one token relative to the pool's current composition, the excess is effectively "donated" — you only receive BPT for the proportional portion. Build your deposit amounts against the **current** pool ratio (re-read pool state right before sending).

## Add instruction

```rust
pub fn add_liquidity<'info>(
    ctx: Context<'_, '_, 'info, 'info, ModifyLiquidity<'info>>,
    token_amounts: Vec<u64>,
    minimum_bpt_amount: u64,
) -> Result<()>
```

Required accounts:

| Account | Type | Description |
|---|---|---|
| `pool` | `CubicPool` (mut) | The pool account |
| `bpt_mint` | `Mint` (mut) | Pool's BPT mint (PDA) |
| `user_bpt_account` | `TokenAccount` (mut) | User's BPT token account |
| `user` | `Signer` | User providing liquidity |
| `token_program` | `TokenInterface` | Token program for the BPT mint; pool token programs are passed in `remaining_accounts` |

Plus `remaining_accounts` as laid out [above](#remaining_accounts-layout) (user-source variant).

### Example: 3-token pool

```
remaining_accounts = [
    user_sol_account,   // [0] user's SOL (source)
    sol_vault,          // [1] pool's SOL vault (destination)
    user_usdc_account,  // [2]
    usdc_vault,         // [3]
    user_btc_account,   // [4]
    btc_vault,          // [5]
    sol_mint,           // [6]
    usdc_mint,          // [7]
    btc_mint,           // [8]
    sol_token_program,  // [9]
    usdc_token_program, // [10]
    btc_token_program,  // [11]
]
```

## BPT calculation

Use [`@cube/sdk` → `CubicPoolClient.quoteAdd`](../sdk/index.md) or simulate the transaction to get the exact mint amount before broadcasting. The on-chain formula for subsequent deposits:

```
ratio_i    = amount_in_i / lp_accessible_balance_i        // 18-dec fixed point
min_ratio  = min(ratio_0, ratio_1, …, ratio_n)
bpt_amount = bpt_supply × min_ratio / 1e18
```

## Important considerations (add)

- **All amounts must be > 0 on the first deposit** so a pool cannot be seeded with missing token balances.
- **First deposit must mint ≥ 1 000 raw BPT** (prevents pool bricking via dust deposits).
- `swaps_enabled` does **not** affect add — LPs can always add when `pool_enabled = true`.
- Protocol fees in `protocol_fees_owed[i]` are excluded from `lp_accessible_balance`, so depositors don't pay BPT against liquidity that belongs to the protocol.

## Events (add)

| Event | Carries |
|---|---|
| `LiquidityAdded` | user, token amounts deposited, BPT minted, timestamp |
| `PoolStateLog` | full pool state snapshot for backend indexing |

---

# Removing Liquidity

Cube supports **proportional withdrawals** only — you burn BPT and receive all pool tokens back proportionally to your BPT share.

## How proportional remove works

1. User specifies `bpt_amount` to burn and `minimum_token_amounts` for per-token slippage.
2. Withdrawal ratio: `ratio = bpt_amount / bpt_total_supply`.
3. For each token: `amount_out[i] = actual_balance[i] × ratio`.
4. Slippage check: each `amount_out >= minimum_token_amounts[i]`.
5. Tokens are transferred from vaults to user.
6. Actual and virtual balances are updated.
7. BPT is burned from the user's account.

BPT supply is read **before** any CPI to prevent manipulation. Pending protocol fees stay accounted for in `protocol_fees_owed[i]` and are settled separately via `collect_protocol_fees`.

## Remove instruction

```rust
pub fn remove_liquidity<'info>(
    ctx: Context<'_, '_, 'info, 'info, RemoveLiquidity<'info>>,
    bpt_amount: u64,
    minimum_token_amounts: Vec<u64>,
) -> Result<()>
```

Required accounts: same set as add (pool, bpt_mint, user_bpt_account, user, token_program), but `remaining_accounts` follow the **vault-source** variant — `vault_i` first, then `user_token_i`.

### Example: 3-token pool

```
remaining_accounts = [
    sol_vault,          // [0] pool's SOL vault (source)
    user_sol_account,   // [1] user's SOL (destination)
    usdc_vault,         // [2]
    user_usdc_account,  // [3]
    btc_vault,          // [4]
    user_btc_account,   // [5]
    sol_mint,           // [6]
    usdc_mint,          // [7]
    btc_mint,           // [8]
    sol_token_program,  // [9]
    usdc_token_program, // [10]
    btc_token_program,  // [11]
]
```

## Withdrawal calculation

```
ratio        = bpt_amount / bpt_total_supply        // 18-dec fixed point
amount_out_i = actual_balance_i × ratio / 1e18
```

## Important considerations (remove)

- `bpt_amount` must be > 0 and ≤ the user's BPT balance.
- `bpt_amount` must be ≤ `bpt_total_supply`.
- `swaps_enabled` does **not** affect remove.
- If burning your BPT would push `bpt_supply` below `MINIMUM_INITIAL_BPT`, the instruction silently caps the burn at the level that leaves the floor intact — the unused BPT stays in your wallet. Re-call if you actually intended to drain the pool (you can't bring supply to zero).

## Events (remove)

| Event | Carries |
|---|---|
| `LiquidityRemoved` | user, BPT burned, token amounts received, timestamp |
| `PoolStateLog` | full pool state snapshot for backend indexing |
