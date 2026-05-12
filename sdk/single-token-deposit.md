# Single-token deposit

A helper that lets an LP add liquidity to a multi-token Cube pool by
supplying a single token. The helper performs the internal swaps
required to assemble a proportional basket and mints BPT to the user
— all in one transaction.

This is implemented by a dedicated on-chain program and surfaced
through `@cube/sdk` (`CubicPoolClient.singleTokenDeposit` /
`SingleTokenDepositClient`).

## Status

- **Mainnet:** not deployed. The frontend hides the single-token
  deposit flow on mainnet.
- **Devnet:** deployed for testing.

Until the helper ships on mainnet, integrators on mainnet should use
the standard proportional `add_liquidity` path on `cubic_pool`.

## SDK surface

```ts
import { CubicPoolClient } from "@cube/sdk";

const tx = await client.singleTokenDeposit({
  inputMint,            // PublicKey of the token the LP holds
  amountIn,             // bigint, native units of inputMint
  minimumBptOut,        // bigint, slippage floor on minted BPT
  user,                 // PublicKey, the LP wallet
});
```

The SDK returns a signed-ready transaction. Inspect the BPT amount
in the simulated logs before broadcasting if you want to apply
custom slippage logic on top.

## Constraints to be aware of when integrating

- The chosen `inputMint` must be one of the pool's constituent tokens.
- The pool must have non-zero balance in the input token; otherwise
  the helper reverts with `PoolHasZeroBalanceToken`.
- Any rounding dust left over after the deposit is refunded to the
  user in the same transaction.
