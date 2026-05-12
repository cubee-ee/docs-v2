# Adding Liquidity

Cube supports two deposit modes:

1. **Proportional deposit** — you supply all tokens in the pool
   simultaneously, matching the current pool ratio. BPT is minted to you
   proportional to your contribution.
2. **Single-token deposit** (default in the UI) — you supply just ONE
   of the pool's tokens. A helper program splits the input across every
   leg, swaps internally as needed, and mints BPT. See
   [SDK / Single-token deposit](../sdk/single-token-deposit.md).

The UI defaults to single-token mode because it requires no
pre-balancing by the user. Power users who already hold the exact pool
ratio can switch to proportional mode via the toggle on the deposit
page.

---

## How It Works

### First Deposit (Empty Pool)

When a pool has no liquidity (BPT supply = 0):

1. User provides a positive `token_amounts[i]` for every pool token
2. The invariant is calculated from the pool's virtual balances using the weighted product formula: `I = product(balance_i ^ weight_i)`
3. BPT minted equals the invariant value
4. A minimum BPT threshold (`MINIMUM_INITIAL_BPT = 1,000`) is enforced to prevent pool bricking
5. Actual balances are set to the deposited amounts

### Subsequent Deposits

When a pool already has liquidity:

1. User provides `token_amounts` for each token (all must be > 0)
2. For each token, the ratio `amount / lp_accessible_balance` is computed,
   where `lp_accessible_balance = actual_balance - protocol_fees_owed`
3. BPT minted = `bpt_supply * min(ratio_i)` — the minimum ratio across all tokens determines the BPT amount
4. Actual balances increase by the deposited amounts
5. Virtual balances increase proportionally to maintain leverage: `virtual_increase = token_amount * (virtual_balance / actual_balance_before)`

---

## On-Chain Instruction

```rust
pub fn add_liquidity<'info>(
    ctx: Context<'_, '_, 'info, 'info, ModifyLiquidity<'info>>,
    token_amounts: Vec<u64>,
    minimum_bpt_amount: u64,
) -> Result<()>
```

**Required accounts:**

| Account | Type | Description |
|---|---|---|
| `pool` | `CubicPool` (mut) | The pool account |
| `bpt_mint` | `Mint` (mut) | Pool's BPT mint (PDA) |
| `user_bpt_account` | `TokenAccount` (mut) | User's BPT token account |
| `user` | `Signer` | User providing liquidity |
| `token_program` | `TokenInterface` | Token program for the BPT mint; pool token programs are passed in `remaining_accounts` |

**remaining_accounts** (4 per token, in pool token order):

```
[user_token_0, vault_0, user_token_1, vault_1, ..., mint_0, mint_1, ..., token_program_0, token_program_1, ...]
```

| Index Pattern | Account | Description |
|---|---|---|
| `i * 2` | `user_token_i` | User's token account for token i (writable) |
| `i * 2 + 1` | `vault_i` | Pool's vault for token i (writable, validated as ATA) |
| `token_count * 2 + i` | `mint_i` | Mint account for token i (for decimals) |
| `token_count * 3 + i` | `token_program_i` | Must match `pool.token_programs[i]` |

### Example: 3-Token Pool (SOL/USDC/BTC)

```
remaining_accounts = [
    user_sol_account,   // [0] user's SOL
    sol_vault,          // [1] pool's SOL vault
    user_usdc_account,  // [2] user's USDC
    usdc_vault,         // [3] pool's USDC vault
    user_btc_account,   // [4] user's BTC
    btc_vault,          // [5] pool's BTC vault
    sol_mint,           // [6] SOL mint
    usdc_mint,          // [7] USDC mint
    btc_mint,           // [8] BTC mint
    sol_token_program,  // [9] token program for SOL mint
    usdc_token_program, // [10] token program for USDC mint
    btc_token_program,  // [11] token program for BTC mint
]
```

---

## BPT Calculation

BPT amounts (both for first and subsequent deposits) are computed by
the program. Use [`@cube/sdk` → `CubicPoolClient.addLiquidity`](../sdk/index.md)
or simulate the transaction to get the exact mint amount before
broadcasting.

### Subsequent Deposits

```
ratio_i = amount_in_i / lp_accessible_balance_i      // 18-decimal fixed point
min_ratio = min(ratio_0, ratio_1, ..., ratio_n)
bpt_amount = bpt_supply * min_ratio / 1e18
```

The minimum ratio ensures proportional deposits are rewarded fairly. Pending
protocol fees are excluded from the LP claim, so depositors do not receive BPT
against liquidity that belongs to the protocol. If you deposit more of one
token relative to the pool's current composition, the excess is effectively
"donated" (you only receive BPT for the proportional portion).

---

## Leverage Maintenance

When liquidity is added, virtual balances are updated to maintain the leverage ratio:

```
for each token i:
    old_actual = actual_balance[i] - token_amount[i]   // balance before deposit
    virtual_increase = token_amount[i] * virtual_balance[i] / old_actual
    virtual_balance[i] += virtual_increase
```

This ensures the leverage ratio (`virtual / actual`) stays approximately constant through deposits.

---

## Slippage Protection

The `minimum_bpt_amount` parameter protects against front-running or unexpected pool state changes. If the computed BPT is less than this amount, the transaction reverts with `InsufficientBptOut`.

---

## Important Considerations

- **All amounts must be > 0 on the first deposit** so a pool cannot be seeded
  with missing token balances
- **First deposit must mint >= 1,000 raw BPT** (prevents pool bricking via tiny deposits)
- Pool must have `pool_enabled = true`
- `swaps_enabled` does **not** affect add liquidity — LPs can always add when the pool is enabled
- Token accounts are validated for correct owner and mint on-chain

---

## Events Emitted

1. **`LiquidityAdded`** — user, token amounts deposited, BPT minted, timestamp
2. **`PoolStateLog`** — full pool state snapshot for backend indexing
