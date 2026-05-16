# @cubee_ee/sdk

📦 **npm**: <https://www.npmjs.com/package/@cubee_ee/sdk>

A TypeScript client for interacting with Cube pools on Solana.
Designed for integrators (aggregators, wallets, bots) who want quotes,
swap transactions, or pool data without re-implementing the on-chain
math.

The SDK ships fresh Anchor IDLs and handles RPC retries, account
parsing, and PDA derivation for you.

---

## Install

```bash
npm install @cubee_ee/sdk
```

Peer dependencies: `@solana/web3.js`, `@solana/spl-token`,
`@coral-xyz/anchor`.

---

## Quick start

```ts
import { getConfig, CubicPoolClient } from "@cube/sdk";
import { PublicKey } from "@solana/web3.js";

const config = getConfig("mainnet", {
  backendEndpoint: "https://api.cubee.ee",
  slippageHundredthsBps: 30_000, // 3 %
});

const pool = new CubicPoolClient({
  config,
  poolAddress: new PublicKey("…"),
  rpc: { endpoint: config.defaults.rpcEndpoint },
});

// Refresh pool state from chain
const { ok, data } = await pool.sync();
if (!ok) return;

// Get a quote for swapping tokens[0] -> tokens[1]
const quote = await pool.getSwapQuote({
  inputMint: data.tokens[0].mint,
  outputMint: data.tokens[1].mint,
  amountIn: 1_000_000_000n,
});
```

Every public method returns `SdkResult<T>` — `{ ok: true, data }` or
`{ ok: false, error: { code, humanMessage } }`. The SDK never throws
on I/O or parse errors.

---

## Top-level clients

| Client | Purpose |
| --- | --- |
| `CubicPoolClient` | Per-pool quotes, swap / add / remove liquidity, sync state |
| `CubeBackendClient` | List pools, fetch routing splits, time-series stats |
| `PoolFactoryClient` | Discover pools by token pair (read-only) |
| `SingleTokenDepositClient` | LP helper for single-token deposits (devnet only) |

Each client method:

1. Returns `SdkResult<T>` (no throws on expected failures).
2. Validates account ownership and discriminators before parsing.
3. Hides RPC retries / backoff behind the surface.

For the swap-router integration (when you want the backend to split a
swap across multiple Cube pools for best execution) see
[Swap Routing](../integration/swap-routing.md).

---

## Token-2022 support

Cube pools accept both classic SPL Token and Token-2022 mints, and a
single pool can mix both programs. The SDK decodes
`tokens[i].tokenProgram` from the on-chain pool account and threads it
through every builder, so callers can pass mints straight from the
pool — no extra plumbing required.

What this means in practice:

- `buildSwapTx`, `buildAddLiquidityTx`, `buildRemoveLiquidityTx`, and
  `buildSingleTokenDepositTx` derive user ATAs under the correct
  program and emit the right `token_program_i` in
  `remaining_accounts`.
- The BPT mint always uses classic SPL Token. Override via
  `bptTokenProgram` on `buildInitializeCubicPoolIx` if a non-default
  program is required.
- Extensions that change transfer amounts (transfer fee, transfer
  hook, confidential transfer) are **not** supported — the AMM math
  expects the vault delta to equal the requested amount, otherwise
  the program reverts with `BalanceMismatch`.

---

## Network configuration

```ts
import { getConfig } from "@cube/sdk";

const cfg = getConfig("mainnet", {
  backendEndpoint: "https://api.cubee.ee",
});
```

`getConfig` returns the program IDs, default RPC, and slippage
defaults for the chosen network. Override any field through the
second argument.

---

## When NOT to use the SDK

- If you only need TVL / volume / fee data, hit the DefiLlama API
  (the protocol is listed under `cube`).
- If you only need pool addresses + token pairs, the backend's
  `/api/pools/by-pair` REST endpoint is enough — see
  [API Reference](../integration/api-reference.md).

For everything else (composing transactions, decoding events,
streaming pool state), use the SDK rather than re-deriving accounts
from the IDL by hand.
