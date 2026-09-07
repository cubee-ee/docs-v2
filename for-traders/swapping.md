# Swapping

The pool supports **exact-input** swaps: you provide a gross input amount and a minimum acceptable output. The contract calculates a weighted AMM output, subtracts any applicable output-token dynamic fee, and checks the minimum against what you will actually receive.

This page matches contracts `audit-fixes-excluded-SF` at `96a2ee2` and SDK `0.11.1` at `27de819`.

## Execution order

1. Check that the pool and swaps are enabled, input is positive, and token indices are distinct and in range.
2. Check the input token is active. An inactive token cannot be sold into the pool, but it can still be bought as output if liquidity permits.
3. Validate mints, stored per-token token programs, user token-account owners and the pool vault addresses.
4. Check the input token's [max-selloff window](max-selloff.md) against **gross input**, using its virtual-balance snapshot and the chain clock.
5. Charge the base fee in input tokens, rounding up. Compute the protocol's share of that fee, also rounding up.
6. Calculate gross output from the weighted curve using input after the base fee and the stored virtual balances.
7. Require gross output to fit the stored **LP actual output reserve**. This reserve already excludes protocol fees.
8. Calculate the input policy's [dynamic fee](dynamic-fee.md), charged in output tokens using four segments above the window threshold.
9. Require `gross_output − dynamic_fee >= minimum_amount_out`.
10. Update virtual/actual balances, fee counters and the accepted selloff state; transfer the full input from the user and the net output to the user.
11. Emit the applicable window event, swap event and balance snapshot.

The transaction is atomic. A failed transfer or later instruction rolls back the swap and its state changes. A failed transaction that lands on-chain can still cost a network transaction fee; a simulation-only failure does not execute token transfers.

## Instruction and accounts

```rust
pub fn swap(
    ctx: Context<Swap>,
    amount_in: u64,
    minimum_amount_out: u64,
    token_in_index: u8,
    token_out_index: u8,
) -> Result<()>
```

Amounts are in the respective mint's raw units. This instruction has no exact-output mode and no separate user-supplied dynamic-fee limit; the net output minimum protects the total result.

| Account | Access | Requirement |
|---|---|---|
| `pool` | Writable | Pool account |
| `token_mint_in`, `token_mint_out` | Read | Match the selected token slots |
| `user_token_account_in`, `user_token_account_out` | Writable | Match the mints and belong to the signing user |
| `vault_in`, `vault_out` | Writable | Correct pool ATAs for the mint and its token program |
| `user` | Signer | Authorizes input transfer |
| `token_program_in`, `token_program_out` | Read | Match each token's program stored by the pool |

There are no remaining-account groups on the direct swap instruction. A pool can mix classic SPL Token and Token-2022 mints, but accepted extension policy does not imply every token can use this runtime transfer path. Use the SDK's compatibility checks and the [token policy documentation](../overview/pool-parameters.md).

## Pricing and fee amounts

The high-level curve is:

```text
curve_input  = amount_in − ceil(amount_in × swap_fee_rate / 1,000,000)
gross_output = virtual_out × [1 − (virtual_in / (virtual_in + curve_input))^(weight_in / weight_out)]
user_output  = gross_output − surge_fee_amount
```

The actual implementation uses fixed-point log/exp and deliberate integer rounding. See [Pricing and Liquidity Math](../technical/math.md); do not use this real-number expression as a byte-exact quote.

Three distinct fee amounts can appear:

| Fee | Denomination | Destination |
|---|---|---|
| Base fee | Input token | Split between LP reserves and the protocol |
| Protocol share of base fee | Input token | A portion of the base fee, not an additional charge on top of it |
| Dynamic/surge fee | Output token | Entirely reserved for the protocol |

The base-fee range is 0–10%, encoded with `1,000,000 = 100%`. The protocol share range is 0–50% of that fee, encoded with `10,000 = 100%`. Dynamic rate points have their own `10,000` scale and can reach 100% within the taxed portion of output. Read the configured pool policy; a pool's base fee alone does not describe its total trading cost.

The accounting is:

```text
input actual/virtual  += gross_input − protocol_input_fee
input protocol owed  += protocol_input_fee
output actual/virtual -= gross_output
output protocol owed += surge_output_fee
```

Both protocol claims remain in the vault until collection. Collection removes those reserved amounts and clears the counters, without reducing LP actual or virtual reserves. Do not price against `actual_balance − protocol_fees_owed`.

## Quote and slippage in the SDK

Call `sync()` before `quoteSwap`. Example using an existing configured `CubicPoolClient`:

```typescript
import BN from "bn.js";

const synced = await client.sync();
if (!synced.ok) throw new Error(synced.error.humanMessage);

// 1 token with 6 decimals; slippage 5,000 / 1,000,000 = 0.5%.
const quote = client.quoteSwap(0, 1, new BN("1000000"), 5_000);
if (!quote.ok) throw new Error(quote.error.humanMessage);

console.log({
  userReceives: quote.data.amountOut.toString(),
  grossOutput: quote.data.grossAmountOut?.toString(),
  inputBaseFee: quote.data.feeAmount.toString(),
  inputProtocolFee: quote.data.protocolFeeAmount.toString(),
  outputSurgeFee: quote.data.surgeFeeAmount?.toString(),
  minimumOutput: quote.data.minAmountOut.toString(),
});
```

Pass the resulting `minAmountOut` when building the swap. It is computed from output **after** surge. A minimum based on gross output can unnecessarily revert; a zero minimum permits a zero-output result and should not be used as ordinary user slippage protection.

The optional fifth quote argument, `nowSeconds`, overrides the chain timestamp saved by `sync()`. Quotes do not reserve balances, freeze admin policy, or update the shared cache. Use fresh state close to submission and keep the output minimum on-chain even when an earlier quote succeeded.

## Price impact and window failures

Curve impact depends on trade size relative to virtual balances and on the weight ratio. Actual output reserves can still limit a trade even when virtual depth is large. The human-unit marginal price in **output per input** is:

```text
virtual_out × weight_in / (virtual_in × weight_out)
  × 10^(decimals_in − decimals_out)
```

The SDK `priceImpactHbps` compares base-fee-adjusted spot output with **net output after surge**, so it includes surge's effect. Display the fee separately if you want to explain why output falls as the window fills.

A `MaxSelloffExceeded` result means the gross input does not fit the candidate rolling window at the execution timestamp. A smaller input or another eligible pool may fit. Waiting can restore headroom as the previous bucket decays and windows rotate, but no particular route or retry is guaranteed to be available. See [Max-Selloff Window](max-selloff.md) for snapshot rebasing, LP rescaling and exact headroom calculation.

## Multi-pool routes and single-token deposits

A route consists of one or more individual swaps. Each leg has its own reserves, base fee, active-token checks and selloff/dynamic-fee policy. Route outputs must include all those effects and enforce the relevant minimums. SDK contract support does not by itself prove that a deployed routing backend uses the same revision. See [Swap Routing](../integration/swap-routing.md) for the backend interface and its boundaries.

Single-token deposits also perform swap legs, so they consume the input token's selloff headroom and can pay surge. The helper uses a positive final BPT minimum for the whole operation rather than positive per-leg token minima. See [Single-Token Deposit](../sdk/single-token-deposit.md).

## Events

- `MaxSelloffWindowAdvanced`: emitted when the input limiter is enabled; contains the accepted effective volume, resolved cap, snapshot and bucket state.
- `Swap`: includes gross `amount_in`, net `amount_out`, input-token `fee_amount` and `protocol_fee_amount`, output-token `surge_fee_amount`, token mints, pool, user and timestamp.
- `PoolStateLog`: contains tracked virtual balances, actual balances and protocol-fee counters after the swap.

The `Swap` event's gross curve output can be recovered as `amount_out + surge_fee_amount`. Do not treat the output fee as an input-token amount or subtract it a second time from event output. See [Tracking Pool Activity](../for-lps/tracking-pool-activity.md).

## Sources

- [Swap instruction, fee rounding and events](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/swap.rs)
- [Protocol-fee collection](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/collect_protocol_fees.rs)
- [SDK quote implementation](https://github.com/coffer-so/sdk/blob/27de819c469056bfb7cd3ab3a4cfdbde741db2f8/src/clients/CubicPoolClient.ts)
