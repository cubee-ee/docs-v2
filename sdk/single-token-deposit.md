# Single-token deposit

> 🚧 **In development — not in production yet.** This module is currently under audit. The on-chain program (`single_token_liquidity`) is deployed on devnet but **not on mainnet**, and the frontend hides single-token deposit on mainnet. Integrators should treat this page as a forward-looking design doc — interfaces may shift before launch.
>
> For mainnet today, use the standard proportional `add_liquidity` path on `cubic_pool`. See [Liquidity](../for-lps/liquidity.md).

A helper that lets an LP add liquidity to a multi-token Cube pool by
supplying a single token. The helper performs the internal swaps
required to assemble a proportional basket and mints BPT to the user
— all in one transaction.

Implemented by a dedicated on-chain program and surfaced through
`@cube/sdk` (`CubicPoolClient.buildSingleTokenDepositTx` /
`SingleTokenDepositClient`).

## Status

- **Mainnet:** not deployed (in audit). Frontend hides the flow.
- **Devnet:** deployed for testing.

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
