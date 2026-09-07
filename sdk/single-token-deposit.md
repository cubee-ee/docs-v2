# Single-token deposits

A single-token deposit, also called a zap, converts one pool token into a proportional deposit basket and mints BPT in one program call. Coffer's `single_token_liquidity` program is included in the mainnet configuration at `7BpdUH1tzTSXLuQNo6YpjJ8Eagw8AkrS6cnkxiJdCFS2`. This page describes the deployed contract revision `96a2ee2` and SDK 0.11.1.

The input must already be one of the pool's tokens. The helper swaps within that same pool; it does not route an unrelated mint through other pools. The pool must be seeded, enabled, and permit swaps. Every internal swap requires its input token to be active. If the input is the sole token with a nonzero actual reserve, the helper makes no internal swaps and does not independently enforce that activation flag. The supported pool size is two to ten tokens.

## Quote the complete operation

```ts
import BN from "bn.js";
import { applySlippage } from "@cubee_ee/sdk";

const synced = await pool.sync();
if (!synced.ok) throw new Error(synced.error.humanMessage);

const quote = pool.quoteSingleTokenDeposit(
  tokenInIndex,
  new BN(amountInRaw),
  5_000,             // 0.5%; hundredths of a basis point
  undefined,         // use the Clock timestamp cached by sync()
  helperBalances,    // optional BN[], in pool-token order
);
if (!quote.ok) throw new Error(quote.error.humanMessage);

const minimumBptAmount = new BN(
  applySlippage(BigInt(quote.data.estimatedBpt.toString()), 5_000).toString(),
);
const built = pool.buildSingleTokenDepositTxs({
  user: walletPublicKey,
  tokenInIndex,
  amountIn: quote.data.amountIn,
  minimumBptAmount,
});
if (!built.ok) throw new Error(built.error.humanMessage);
// Send and confirm built.data.setup first when present.
// Then compile built.data.deposit with the pool ALT and obtain signatures.
```

Here `pool` is a synced `CubicPoolClient`; the other identifiers are supplied by the application. `helperBalances` means the helper's **existing token ATA balances before this deposit**, one `BN` per pool token. Omit it only when quoting an empty helper. `sync()` does not fetch those ATA balances for you. Obtain their addresses from `helperPda()` and each mint's token program, read their current raw balances, and pass the complete vector. Missing ATAs have zero balances. An existing helper BPT balance is not part of this vector and is not included in newly minted BPT.

The standalone `SingleTokenDepositClient` exposes the same quote through `quote(tokenInIndex, amountIn, slippage?, nowSeconds?, helperBalances?)`. `pool.singleTokenDeposit` is a client getter, not a function.

| Quote field | Meaning |
| --- | --- |
| `tokenInIndex`, `amountIn` | Input token index and raw amount |
| `allocations` | Input amount allocated to each leg; entries sum to `amountIn` |
| `expectedOuts` | Estimated outputs of the internal swaps; the input slot has no swap output |
| `minOuts` | Informational per-leg slippage estimates; these are not instruction arguments |
| `depositedAmounts` | Estimated token amounts actually credited to the pool after proportional cropping |
| `refundAmounts` | Estimated helper token balances returned to the user |
| `estimatedBpt` | Estimated newly minted BPT before applying the user's floor |
| `sidelinedTokenIndices` | Tokens whose pool actual balance is zero and which are excluded from allocation |

## Allocation, fees, and slippage

Allocation uses each token's weight multiplied by `min(actualBalance, virtualBalance) / virtualBalance`, with the contract's integer truncation. The helper keeps the input-token allocation and swaps the other allocations in token order. Each leg changes the reserves and the input token's sell-off window seen by the next leg. SDK 0.11.1 simulates that sequence, including base swap fees, protocol shares, the dynamic output fee, the sell-off cap, and the final proportional join.

The quote validates the same arithmetic bounds used by these operations and rejects unusable amounts. Small allocations can truncate to zero. A pool with zero actual balance for the input token cannot accept this route. A route can also fail its cumulative sell-off limit even if an individual leg would pass in isolation. Details and scales are in [pool mathematics](../technical/math.md), [sell-off limits](../for-traders/max-selloff.md), and [dynamic fees](../for-traders/dynamic-fee.md).

There are only three arguments on the deposit instruction: `amount_in`, `token_in_index`, and `minimum_bpt_amount`. Internal swaps use zero per-leg output floors. The **positive final BPT floor** protects the complete swap-and-join operation. Passing `slippageHundredthsBps` to a quote does not put that number on-chain. Build with an explicit, positive `minimumBptAmount`; the SDK rejects an omitted or zero floor.

Existing tokens in helper ATAs can affect the basket and refunds. Include them in the quote instead of assuming they disappear. Because reserves, windows, and helper balances can change after reading, a quote is an estimate at the specified state and time. The signed BPT floor remains the execution constraint.

## Setup, account size, and signing

`buildSingleTokenDepositTx` combines ATA setup with the deposit. `buildSingleTokenDepositTxs` returns `{ setup, deposit }`: setup contains idempotent ATA-creation instructions, and deposit contains the swap-and-join operation without those setup instructions. Use the split form when the combined message is too large. Confirm setup before sending deposit.

Splitting does not create the pool ALT. Initialize the ALT through the authorized pool or protocol-admin builder, confirm it, and wait for its entries to become usable in a subsequent slot. Then use `compileBuiltTx` with the synced pool or `buildVersionedTx` with its ALT address. The caller must supply signatures, account for transaction-size and compute limits, send, and confirm. These methods do not manage a WalletConnect or Ledger session.

Underlying token programs and the BPT token program come from mint ownership. Compatible classic SPL Token and Token-2022 mints are supported, but the deployed program does not implement arbitrary Token-2022 transfer extensions. The STLD quote checks active reserve legs, the input, and nonzero existing helper balances. The instruction builder checks **all pool mints**, including sidelined tokens, because refunds can touch pre-existing helper balances. A quote can succeed while the stricter builder rejects an incompatible sidelined mint. See [SDK token compatibility](index.md#supported-tokens-and-abi-access).

## Interpreting confirmed events

The `SingleTokenDeposit` event records the basket passed to the pool join. The pool may crop that basket again. For actual token amounts credited, use the corresponding `LiquidityAdded` event from the successful transaction. Do not treat every event field named `deposited_amounts` as a post-crop wallet debit. The quote's `depositedAmounts` estimates the actual credited amounts; its refunds include remaining helper tokens.

Use `parseContractEvents` for the complete current event ABI and verify the successful transaction and emitting program before indexing. `parseCubicPoolEvents` provides compatibility naming and value conversions, with a fallback for other current events. Use the generic typed API when retaining exact IDL field names and integer representations matters.
