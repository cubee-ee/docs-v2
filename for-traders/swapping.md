# Swapping

Cube supports **EXACT_IN** swaps — you specify the amount of input token, and the protocol computes the output.

---

## How a Swap Works

1. User specifies: `amount_in`, `minimum_amount_out`, `token_in_index`, `token_out_index`
2. The program validates the pool is enabled, swaps are enabled, and token indices are valid
3. Swap fee is deducted from the input: `fee = floor(amount_in * swap_fee_rate / 1,000,000)`
4. Protocol fee is computed from the swap fee: `protocol_fee = floor(fee * protocol_fee_rate / 10,000)`
5. LP-accessible balances are computed by excluding `protocol_fees_owed`
6. Output is calculated using the weighted AMM formula on the scaled virtual balances
7. The result must be <= LP-accessible output balance or the swap reverts
8. Slippage check: `amount_out >= minimum_amount_out`
9. Tokens are transferred: input from user to vault, output from vault to user
10. Balances and invariant are updated

### Swap Formula

```
amountOut = virtualBalanceOut * (1 - (virtualBalanceIn / (virtualBalanceIn + amountInAfterFee)) ^ (weightIn / weightOut))
```

Where:
- `virtualBalanceIn`, `virtualBalanceOut` — virtual balances of the respective tokens
- `amountInAfterFee` — input amount minus swap fee
- `weightIn`, `weightOut` — token weights (basis points, converted to 18-decimal fixed point for calculation)
- Output is checked against `actualBalanceOut - protocolFeesOwedOut`

### On-Chain Instruction

```rust
pub fn swap(
    ctx: Context<Swap>,
    amount_in: u64,
    minimum_amount_out: u64,
    token_in_index: u8,
    token_out_index: u8,
) -> Result<()>
```

**Required accounts:**

| Account | Type | Description |
|---|---|---|
| `pool` | `CubicPool` (mut) | The pool account |
| `token_mint_in` | `Mint` | Input token mint |
| `token_mint_out` | `Mint` | Output token mint |
| `user_token_account_in` | `TokenAccount` (mut) | User's input token account |
| `user_token_account_out` | `TokenAccount` (mut) | User's output token account |
| `vault_in` | `TokenAccount` (mut) | Pool's input token vault |
| `vault_out` | `TokenAccount` (mut) | Pool's output token vault |
| `user` | `Signer` | User signing the transaction |
| `token_program_in` | `TokenInterface` | Must match the input token's stored program |
| `token_program_out` | `TokenInterface` | Must match the output token's stored program |

---

## Price Impact

Price impact depends on:

1. **Trade size relative to virtual balances** — larger trades move the price more
2. **Token weights** — swapping into a low-weight token causes more impact
3. **Leverage** — higher leverage (virtual >> actual) gives tighter spreads for small trades but the same actual balance constraint

For small trades, the price is approximately the spot price:

```
spotPrice = (virtualBalanceIn / weightIn) / (virtualBalanceOut / weightOut)
```

---

## Slippage Protection

The `minimum_amount_out` parameter protects against excessive slippage. If the calculated output is less than this amount, the transaction reverts with `SlippageExceeded`.

Set this value based on your acceptable price tolerance — typically 0.5%–1% below the expected output from the swap route API.

---

## Fee Mechanics

### Swap Fee

- Charged on the **input** token
- Range: 0% to 10% (stored as `u32` in hundredths of a basis point — `10,000` = 1%, max `100,000` = 10%)
- The fee stays in the pool vault, increasing actual balances for LPs
- Zero-fee swaps are rejected when `swap_fee_rate > 0` (prevents dust micro-swaps)

### Protocol Fee

- A portion of the swap fee reserved for the protocol
- Range: 0% to 50% of the swap fee (stored as `u16`, where 5,000 = 50%)
- Default: 20% of the swap fee
- Tracked in `protocol_fees_owed` per token, collected separately by the protocol authority
- Protocol fees remain in `actual_balances` until collection; swaps and LP
  withdrawals exclude them from LP-accessible liquidity
- When collected, the contract decreases both actual and virtual balances
  proportionally so leverage is preserved

### Fee Calculation Example

```
Input: 1,000,000 SOL lamports
Swap fee rate: 3,000 (= 0.3%)
Protocol fee rate: 2,000 (= 20% of swap fee)

swap_fee = floor(1,000,000 * 3,000 / 1,000,000) = 3,000 lamports
protocol_fee = floor(3,000 * 2,000 / 10,000) = 600 lamports
amount_in_after_fee = 1,000,000 - 3,000 = 997,000 lamports

LP revenue per swap = swap_fee - protocol_fee = 2,400 lamports
```

---

## Swap Routing (Multi-Pool)

If the same token pair exists across multiple pools, the Cube backend splits
the swap across pools using the same BigInt quote math as the SDK and
contract. Candidate routes are built from fresh on-chain state, exclude
protocol-fee reserves from LP liquidity, and are skipped if the contract would
reject them.

See [Swap Routing](../integration/swap-routing.md) for details on the routing algorithm.

### Using the Swap Route API

Before submitting a swap transaction, query the backend for the optimal route:

```
GET https://api.cubee.ee/api/pools/swap-route?tokenIn=<mint>&tokenOut=<mint>&amountIn=<amount>
```

The response includes per-pool split amounts, expected outputs, and vault addresses needed to build the transaction.

---

## Events Emitted

Each swap emits two events:

1. **`Swap`** — trade details (pool, user, tokens, amounts, fees, timestamp)
2. **`PoolStateLog`** — full pool state snapshot for backend indexing
