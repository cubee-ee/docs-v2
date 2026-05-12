# Smart Contracts

Reference for the parts of the Cube on-chain programs that
integrators (AMM bots, swap aggregators, indexers) need to interact
with.

The full Anchor IDL is published alongside the DefiLlama TVL adapter
and contains every instruction discriminator, account layout, and
event the program emits — use that as the source of truth for code
generation.

---

## Program IDs (mainnet)

| Program | ID | Status |
| --- | --- | --- |
| Cubic Pool | `8iQtGj9mcUfFUGaiCpPy89swC3s8YTC8FhVZWfgeZhwu` | Deployed |
| Protocol Fees Authority | `3jiojHZbjJQ7QLMGSTjFwxVEmx4NtuRy34nLAmsJME81` | Deployed |
| Single Token Liquidity | `7BpdUH1tzTSXLuQNo6YpjJ8Eagw8AkrS6cnkxiJdCFS2` | Devnet only |

---

## What integrators interact with

Only three instructions are relevant for routing / aggregation:

- `swap` — execute a swap
- `add_liquidity` — proportional deposit (LP-facing, useful for vault integrations)
- `remove_liquidity` — proportional withdrawal

All other instructions are governance / admin operations gated by the
treasury PDA and are not callable by integrators.

---

## `swap`

Executes a swap on a single pool.

```rust
pub fn swap(
    ctx: Context<Swap>,
    amount_in: u64,
    minimum_amount_out: u64,
    token_in_index: u8,
    token_out_index: u8,
) -> Result<()>
```

| Argument | Description |
| --- | --- |
| `amount_in` | Input amount in native token units |
| `minimum_amount_out` | Slippage floor — transaction reverts if output would be less |
| `token_in_index` / `token_out_index` | 0-based indices into the pool's token list |

### Accounts

| Name | Notes |
| --- | --- |
| `pool` (mut) | The pool PDA |
| `token_mint_in` / `token_mint_out` | Must match the pool's token at the given index |
| `user_token_account_in` (mut) / `user_token_account_out` (mut) | Standard token accounts owned by the user |
| `vault_in` (mut) / `vault_out` (mut) | ATA of the pool over the corresponding mint and token program |
| `user` (signer) | Trader / router |
| `token_program_in` / `token_program_out` | SPL Token or Token-2022, matching each side's mint |

`vault_in` and `vault_out` are deterministically derived as
`ATA(pool, mint, token_program)`. The token program for each side is
stored on the pool account; read it before composing the transaction.

### Behaviour

- **EXACT_IN.** The program computes the exact output amount from
  `amount_in` and the current pool state and pays it out — see
  [Pricing Model](math.md) for the formula.
- A swap fee (set per pool) is deducted from `amount_in` before the
  swap formula is evaluated. The fee stays in the pool; a share of
  it is set aside for the protocol.
- The transaction reverts on insufficient output (`minimum_amount_out`
  not met), if the pool is disabled, or if the computed output
  exceeds the pool's available balance.

### Events

`Swap` is emitted on success. Fields include input/output mints,
amounts, the user, the swap fee, the protocol fee, and a timestamp.
Indexers / dashboards should subscribe to this event.

---

## `add_liquidity` / `remove_liquidity`

Proportional join and exit. Used by vault integrations and the
single-token-deposit helper (see
[SDK / Single-token deposit](../sdk/single-token-deposit.md)).

Integrators typically don't compose these by hand — use
`@cube/sdk` (`CubicPoolClient.addLiquidity`, `removeLiquidity`),
which derives accounts and minimum-out calculations from the
current pool state.

---

## Pool discovery

To list all pools:

```ts
import { Connection, PublicKey } from "@solana/web3.js";

const accounts = await connection.getProgramAccounts(
  new PublicKey("8iQtGj9mcUfFUGaiCpPy89swC3s8YTC8FhVZWfgeZhwu"),
  // Anchor's 8-byte discriminator filter is the standard approach;
  // see the IDL for the CubicPool account schema.
);
```

Each `CubicPool` account encodes the pool's tokens, weights, swap fee
rate, and the balances used for pricing. Decode it via the IDL
(`coral-xyz/anchor`'s `Program.account.cubicPool.all()` or any Anchor
parser of your choice).

The backend at `https://api.cubee.ee` also exposes a pool index over
REST — see [API Reference](../integration/api-reference.md).
