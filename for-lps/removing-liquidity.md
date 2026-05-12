# Removing Liquidity

Cube supports **proportional withdrawals** — you burn BPT and receive all pool tokens proportionally based on your BPT share.

---

## How It Works

1. User specifies `bpt_amount` to burn and `minimum_token_amounts` for slippage protection
2. The withdrawal ratio is computed: `ratio = bpt_amount / bpt_total_supply`
3. For each token: `amount_out[i] = actual_balance[i] * ratio`
4. Slippage check: each `amount_out >= minimum_token_amount`
5. Tokens are transferred from vaults to user
6. Actual and virtual balances are updated
7. BPT is burned from user's account

---

## On-Chain Instruction

```rust
pub fn remove_liquidity<'info>(
    ctx: Context<'_, '_, 'info, 'info, RemoveLiquidity<'info>>,
    bpt_amount: u64,
    minimum_token_amounts: Vec<u64>,
) -> Result<()>
```

**Required accounts:**

| Account | Type | Description |
|---|---|---|
| `pool` | `CubicPool` (mut) | The pool account |
| `bpt_mint` | `Mint` (mut) | Pool's BPT mint (PDA) |
| `user_bpt_account` | `TokenAccount` (mut) | User's BPT token account |
| `user` | `Signer` | User removing liquidity |
| `token_program` | `TokenInterface` | Token program for burning BPT; pool token programs are passed in `remaining_accounts` |

**remaining_accounts** (4 per token, in pool token order):

```
[vault_0, user_token_0, vault_1, user_token_1, ..., mint_0, mint_1, ..., token_program_0, token_program_1, ...]
```

> **Note:** The order is **vault first, then user account** — opposite from add_liquidity. This follows the flow direction: source (vault) before destination (user).

| Index Pattern | Account | Description |
|---|---|---|
| `i * 2` | `vault_i` | Pool's vault for token i (writable, source) |
| `i * 2 + 1` | `user_token_i` | User's token account for token i (writable, destination) |
| `token_count * 2 + i` | `mint_i` | Mint account for token i (for decimals) |
| `token_count * 3 + i` | `token_program_i` | Must match `pool.token_programs[i]` |

### Example: 3-Token Pool (SOL/USDC/BTC)

```
remaining_accounts = [
    sol_vault,          // [0] pool's SOL vault (source)
    user_sol_account,   // [1] user's SOL (destination)
    usdc_vault,         // [2] pool's USDC vault
    user_usdc_account,  // [3] user's USDC
    btc_vault,          // [4] pool's BTC vault
    user_btc_account,   // [5] user's BTC
    sol_mint,           // [6] SOL mint
    usdc_mint,          // [7] USDC mint
    btc_mint,           // [8] BTC mint
    sol_token_program,  // [9] token program for SOL mint
    usdc_token_program, // [10] token program for USDC mint
    btc_token_program,  // [11] token program for BTC mint
]
```

---

## Withdrawal Calculation

```
ratio = bpt_amount / bpt_total_supply     // 18-decimal fixed point

for each token i:
    amount_out[i] = actual_balance[i] * ratio / 1e18
```

The proportional withdrawal uses the raw `actual_balance` directly. Pending protocol fees stay accounted for in `pool.protocol_fees_owed[i]` and are settled separately through `collect_protocol_fees`.

---

## Leverage Maintenance

When liquidity is removed, virtual balances are decreased to maintain the leverage ratio:

```
for each token i:
    old_actual = actual_balance[i]                    // balance before the bookkeeping subtraction
    virtual_decrease = amount_out[i] * virtual_balance[i] / old_actual
    virtual_balance[i] -= virtual_decrease   // saturating
```

`old_actual` here is the snapshot read at the top of the handler (before
`actual_balances[i] -= amount_out[i]`). This preserves the `virtual / actual`
ratio through withdrawals.

---

## Slippage Protection

The `minimum_token_amounts` vector (one per token) protects against:
- Front-running that drains pool balances
- Large trades between your quote and execution
- Unexpected state changes

If any token's output is less than its minimum, the transaction reverts with `InsufficientTokensOut`.

---

## Important Considerations

- `bpt_amount` must be > 0 and <= the user's BPT balance
- `bpt_amount` must be <= `bpt_total_supply`
- BPT supply is read **before** any CPI to prevent manipulation
- Pool must have `pool_enabled = true`
- `swaps_enabled` does **not** affect remove liquidity
- Vault accounts are validated by deriving the expected ATA at runtime
- User token accounts are validated for correct owner and mint

---

## Events Emitted

1. **`LiquidityRemoved`** — user, BPT burned, token amounts received, timestamp
2. **`PoolStateLog`** — full pool state snapshot for backend indexing
