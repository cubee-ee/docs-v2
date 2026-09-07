# Liquidity: seed, add and remove

A direct deposit or withdrawal operates on LP-owned `actual_balance`, not on
`actual_balance - protocol_fees_owed`. Protocol fees are a separate bucket.
All amounts below are raw token units; BPT has 9 decimals. Use `BN` for SDK
instruction amounts and `bigint` for the pure math helpers.

## Which operation to use

| Operation | Preconditions | Result |
| --- | --- | --- |
| First/seed deposit | BPT supply is zero; signer is the nonzero current pool admin; pool enabled | Takes the supplied basket verbatim and mints invariant-based BPT |
| Subsequent proportional add | Pool enabled; supply positive; live/sidelined amounts match the state | Takes a proportional basket within the supplied spend ceilings |
| Proportional remove | Pool enabled; positive valid BPT request | Burns an effective BPT amount and pays proportional actual reserves |
| Single-token deposit | Seeded pool and positive actual input reserve; pool and swaps enabled | Helper swaps internally, joins the pool, forwards BPT and refunds excess tokens |

`swaps_enabled` and `is_active` do not gate direct add/remove. A single-token
route requires `swaps_enabled`, and every internal swap must pass its input
activation flag and selloff limit. Admin and token-program checks still apply.
An enabled pool is therefore necessary but not sufficient for an operation to
succeed.

## First deposit

The current pool admin must sign the seed. Disabling the pool admin before
seeding prevents this path. At least one supplied token amount must be positive;
other slots may remain sidelined with zero actual balance.

The seed takes the supplied amounts without proportional cropping and sets the
actual balances to those amounts. Initial BPT is computed by the contract's
weighted invariant from virtual balances, with token decimals normalized to six
places. The invariant is returned as a raw BPT amount; it must fit `u64` and be
at least **1,000 raw BPT**. The implementation has an all-virtual-balances-zero
fallback using the supplied amounts, although normal initialization requires
positive virtual balances.

Use `quoteSeedDeposit(user, tokenAmounts, slippageHundredthsBps?)`, then pass its
`minimumBptAmount` to `buildAddLiquidityTx`. This quote verifies the cached
current pool admin and reports the full basket, zero refunds and `limitingTokenIndex = -1`.
The amount of first BPT is not a USD valuation of the deposit.

## Subsequent proportional deposits

`token_amounts` is a **spend ceiling vector**, one value per pool token. For each
slot with positive actual balance, the supplied ceiling must be positive. A
slot with zero actual balance must be offered zero; normal add-liquidity cannot
revive it. A successful swap with that token as input can revive it instead.

For fixed-point scale `Q = 10^18`, old LP balances `A[i]`, old BPT supply `S`
and supplied ceilings `C[i]`:

```text
r[i]       = floor(C[i] * Q / A[i])               for A[i] > 0
r          = min(r[i])
bptOut     = floor(S * r / Q)
deposit[i] = floor(A[i] * r / Q)
left[i]    = C[i] - deposit[i]
```

Only `deposit[i]` leaves the user's wallet. The unused `left[i]` stays there;
it is not donated and there is no refund transfer for a direct add. The SDK
calls this unused portion `refundAmounts`.

Example in raw units: actual `[1,000,000, 3,000,000]`, ceilings
`[150,000, 300,000]` and supply `1,000,000,000` yield ratio 0.1, actual deposits
`[100,000, 300,000]`, unused amounts `[50,000, 0]` and `100,000,000` new BPT.

The operation must mint positive BPT and meet `minimum_bpt_amount`. The normal
SDK builder requires a positive minimum, even though the raw contract argument
can be zero. `quoteAddLiquidity(ceilings, slippageHundredthsBps?)` returns the
cropped amounts, unused amounts, BPT result, minimum BPT and limiting token.

Actual balances grow by the amounts actually transferred. Every virtual
balance grows by `floor(oldVirtual[i] * r / Q)`, including sidelined slots.
Enabled selloff-window buckets and their virtual-balance snapshots are rescaled
by the same ratio. These are integer operations, so do not assume the final
virtual/actual ratios are mathematically identical at raw-unit precision.

### Rounding limit in this contract version

This revision computes BPT from the ceiling ratio before rounding each actual
transfer. At very small raw balances, a live transfer can round to zero while
BPT is positive. That can dilute existing LP shares without matching reserve
funding. It is a contract limitation, not a protocol-fee charge.

SDK 0.11.1 quotes and normal add-liquidity builders reject a proportional
basket with a zero rounded live-token leg. Generic instruction construction
does not enforce that SDK policy, and client checks do not repair the contract.
The exact integer formulas above are documented for compatibility; they are not
a claim that every permitted contract edge case is safe.

## Proportional withdrawals

There is no direct single-token withdrawal instruction. A removal pays the
pool's current basket; conversion to one token requires additional swaps and
has its own fees, capacity and slippage constraints.

Let `B` be requested raw BPT, `S` the pre-burn supply and `Q = 10^18`:

```text
require 0 < B <= S
burn       = min(B, S - 1,000)                   require burn > 0
r          = floor(burn * Q / S)
out[i]     = floor(A[i] * r / Q)
newA[i]    = A[i] - out[i]
newV[i]    = V[i] - floor(V[i] * r / Q)          saturating subtraction
```

The two floors in `r` and `out` matter; replacing them with one final division
can change raw outputs. The user must hold the BPT actually burned. Unburned
BPT stays in the wallet; it is not transferred to a burn address. Repeating a
withdrawal cannot remove the minimum supply floor.

`quoteRemove(requestedBpt)` returns `tokenOuts` and `effectiveBptIn`. Derive
`minimumTokenAmounts` from those outputs, not from an unclamped 100% withdrawal.
The SDK builder requires an explicit floor vector of the correct length; zeros
are allowed but disable protection for those outputs.

Protocol-fee counters are not redeemed by BPT and remain unchanged. Selloff
buckets and snapshots shrink with the burn ratio when the limiter is enabled.
No swap fee or surge fee is charged by direct removal. Token-program, Solana
transaction and account-creation costs are separate concerns.

## Instruction accounts

Both instructions use these five named accounts, with separate Anchor context
structs (`ModifyLiquidity` for add, `RemoveLiquidity` for remove):

| Account | Access | Meaning |
| --- | --- | --- |
| `pool` | Writable | The CubicPool account |
| `bpt_mint` | Writable | Pool-derived BPT mint |
| `user_bpt_account` | Writable | User-owned account for that BPT mint |
| `user` | Signer | Owner supplying tokens or burning BPT |
| `token_program` | Read-only | Actual owner program of the BPT mint |

There must also be exactly `4 * N` remaining accounts:

| Position | Add | Remove |
| --- | --- | --- |
| `2*i` | User token account, writable | Pool vault, writable |
| `2*i + 1` | Pool vault, writable | User token account, writable |
| `2*N + i` | Token mint, read-only | Token mint, read-only |
| `3*N + i` | That mint's token program | That mint's token program |

These are paired accounts followed by two blocks, not `N` groups of four.
Vaults are derived ATAs for `(pool, mint, tokenProgram)`. On add, a zero offered
slot skips user-token-account validation/transfer; the full account list remains
required. On remove, user token accounts are validated even for zero outputs.
BPT's program and each reserve token's program may differ.

Builders return instructions, not signed transactions. For larger pools, compile
with the pool's ALT via `compileBuiltTx`, check transaction size and required
accounts, then use the wallet to sign and send. Reserve-token SOL is wrapped
SOL; direct add/remove builders do not automatically wrap or unwrap it.

## Single-token deposit and events

The deployed single-token helper runs actual swap CPIs, then a proportional
add, BPT forwarding and token refunds in one deposit transaction. Its whole-route
minimum BPT protects the final result. Existing helper balances can affect the
join and refunds. See the [single-token guide](../sdk/single-token-deposit.md)
for allocation math, separate setup, helper balances and the 10-token limit.

`LiquidityAdded` records the transferred basket and minted BPT;
`LiquidityRemoved` records the effective burn and actual outputs. Their
`PoolStateLog` companion contains balance vectors, not every field of the pool.
A helper deposit also emits internal swap/join events plus its final
`SingleTokenDeposit` event. Avoid counting the same economic action twice.

Sources: [add handler](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/add_liquidity.rs),
[remove handler](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/remove_liquidity.rs),
[SDK client](https://github.com/coffer-so/sdk/blob/27de819c469056bfb7cd3ab3a4cfdbde741db2f8/src/clients/CubicPoolClient.ts).
