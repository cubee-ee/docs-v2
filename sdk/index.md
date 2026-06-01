# @cubee_ee/sdk

📦 **npm**: <https://www.npmjs.com/package/@cubee_ee/sdk>

A TypeScript client for interacting with Cube pools on Solana.
Designed for integrators (aggregators, wallets, bots) who want quotes,
swap transactions, or pool data without re-implementing the on-chain
math.

The SDK ships fresh Anchor IDLs and handles RPC retries, account parsing, PDA derivation, and human-readable error mapping for you.

---

## Install

```bash
npm install @cubee_ee/sdk
```

Peer dependencies: `@solana/web3.js`, `@solana/spl-token`, `@coral-xyz/anchor`.

---

> 🔐 **Backend access requires an API key.** Calls into `CubeBackendClient` (list pools, swap-route, stats, leaderboard) need a Cube API key. Pool **on-chain** calls go directly to your RPC and do not require a key.
>
> Request a key:
>
> - Telegram chat: **[@cubee\_chat](https://t.me/cubee_chat)**
> - Direct: **[@sepezho](https://t.me/sepezho)**
>
> Pass it via `getConfig({ apiKey })`. The SDK adds `X-Cube-Api-Key` to every backend request automatically.

---

## The `SdkResult<T>` contract

**Every** public SDK method returns `SdkResult<T>` — never throws on expected failures.

```ts
type SdkResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: SdkErrorCode; humanMessage: string; cause?: unknown } };

type SdkErrorCode =
  | "invalid_input"        // bad arg / out-of-range / decode failure
  | "insufficient_funds"   // not enough balance / pool liquidity
  | "slippage_exceeded"    // amount_out < min_amount_out, or BPT < min
  | "math_overflow"        // u128 saturate, division by zero
  | "pool_disabled"        // master kill switch on
  | "swaps_disabled"       // per-pool swap toggle off
  | "not_found"            // pool / token / endpoint 404
  | "rpc_unavailable"
  | "rpc_rate_limited"
  | "rpc_timeout"
  | "backend_unavailable"
  | "backend_unauthorized" // missing / wrong API key
  | "unknown";
```

Anchor program errors are auto-mapped from the contract IDL to these categories — the SDK is the single place that knows about every cubic-pool error code, you just check `error.code`.

---

## Top-level clients

| Client | Purpose |
| --- | --- |
| `CubicPoolClient` | Per-pool quotes, swap / add / remove liquidity, sync state, parse events |
| `CubeBackendClient` | List pools, fetch routing splits, leaderboard, time-series stats, prices |
| `PoolFactoryClient` | Build pool-creation transactions (`initialize_config`, `initialize_cubic_pool`) |
| `AdminClient` | Treasury + pool-admin operations (initiate/accept/cancel transfer, set fees, enable flags, register tokens, collect protocol fees) |
| `SingleTokenDepositClient` | LP helper for single-token deposits (devnet only — see [SDK / Single-token deposit](single-token-deposit.md)) |

---

## Config

```ts
import { getConfig, type CubeConfig } from "@cubee_ee/sdk";

const cfg = getConfig("mainnet", {
  backendEndpoint: "https://api.cubee.ee",
  apiKey: process.env.CUBE_API_KEY,
  defaults: {
    rpcEndpoint: "https://your-rpc.example",
    commitment: "confirmed",
    slippageHundredthsBps: 30_000,     // 3% default for swaps
  },
});
```

`getConfig(network, overrides?)` returns the program IDs, default RPC, and slippage defaults for the network (`"mainnet"` or `"devnet"`). Any field is overridable.

---

## `CubicPoolClient` — pool operations

The main client. One instance per pool.

### Construction

```ts
import { CubicPoolClient } from "@cubee_ee/sdk";
import { Connection, PublicKey } from "@solana/web3.js";

const connection = new Connection(cfg.defaults.rpcEndpoint, "confirmed");

const client = new CubicPoolClient({
  config: cfg,
  poolAddress: new PublicKey("CSgrEBxsghBsY1oXycBEhZVTd5PEbZuWFDZHHrtFF7yb"),
  connection,
});
```

### `sync(): Promise<SdkResult<PoolInfo>>`

Refresh the in-memory pool snapshot from on-chain state. Most other methods require a fresh sync — call this before every quote / build if you care about up-to-the-block accuracy. Otherwise it's safe to cache for ~1 slot.

```ts
const r = await client.sync();
if (!r.ok) return console.error(r.error.humanMessage);
console.log(`Synced ${r.data.tokenCount}-token pool, TVL ${r.data.tokens.length} legs`);
```

`PoolInfo` mirrors the on-chain `CubicPool` struct — token list (mint, decimals, weight, leverage, vbal, abal), swap fee, protocol fee, flags, range-manager state, lookup table.

### `getCached(): PoolInfo | undefined`

Returns the last `sync()`d state without hitting RPC. Returns `undefined` if you haven't synced yet.

### `quoteSwap(params): SdkResult<SwapQuote>`

Quote a swap WITHOUT building or sending a tx. Pure math.

```ts
const q = client.quoteSwap({
  tokenInIndex: 0,
  tokenOutIndex: 1,
  amountIn: new BN(1_000_000_000),
  slippageHundredthsBps: 50_000, // 5%
});
// SwapQuote: { amountOut, minAmountOut, feeAmount, protocolFeeAmount, priceImpactBps, spotPriceBefore, spotPriceAfter }
```

| Field | Meaning |
|---|---|
| `amountOut` | Expected output (raw units, no slippage) |
| `minAmountOut` | `amountOut × (1 − slippage)` — pass to `buildSwapTx` to enforce |
| `feeAmount` | Swap fee deducted from `amount_in` |
| `protocolFeeAmount` | Share of `feeAmount` going to protocol |
| `priceImpactBps` | Basis-points price impact |
| `spotPriceBefore` / `spotPriceAfter` | Curve spot before and after this trade |

### `quoteAdd(amounts): SdkResult<AddQuote>`

```ts
const r = client.quoteAdd([new BN(...), new BN(...), ...]);
// AddQuote: { bptOut, minBptOut, share, effectiveAmounts }
```

`effectiveAmounts[i]` may differ from your input if you over-supplied one token relative to the pool ratio — the contract takes only the proportional minimum and the rest is "donated" (so the SDK clamps these).

### `quoteRemove(bptIn): SdkResult<{ tokenOuts: BN[] }>`

```ts
const r = client.quoteRemove(new BN(100_000));
// tokenOuts[i] = expected amount of token i back, native units
```

Pass `tokenOuts` (with a slippage haircut) as `minimum_token_amounts` to `buildRemoveLiquidityTx`.

### `quoteSingleTokenDeposit(params): SdkResult<...>` 🚧

Off-chain quote for the single-token helper. **Devnet only** — see [Single-token deposit](single-token-deposit.md) for status.

### `buildSwapTx(params): SdkResult<BuiltTx>`

Build the **instructions** (not signed) for a swap. Pair with the wallet/signer of your choice.

```ts
const r = client.buildSwapTx({
  user: walletPubkey,
  tokenInIndex: 0,
  tokenOutIndex: 1,
  amountIn: new BN(1_000_000_000),
  slippageHundredthsBps: 50_000,
  // OR pass an explicit floor (overrides slippageHundredthsBps):
  // minAmountOut: new BN(...),
});
// BuiltTx: { instructions: TransactionInstruction[]; computeUnits?: number; ... }
```

**Important** (since `0.2.2`): if you omit `minAmountOut` and the internal quote can't run, `buildSwapTx` now **returns an error** instead of falling back to `minAmountOut = 0` (which would have meant "accept any output, including 0"). Catch the error or always pass an explicit floor.

### `buildAddLiquidityTx(params)`

```ts
client.buildAddLiquidityTx({
  user: walletPubkey,
  tokenAmounts: [new BN(...), ...],   // length = token_count
  minimumBptAmount: new BN(...),
});
```

### `buildRemoveLiquidityTx(params)`

```ts
client.buildRemoveLiquidityTx({
  user: walletPubkey,
  bptAmount: new BN(...),
  minimumTokenAmounts: [new BN(...), ...],   // length = token_count
});
```

### `buildSingleTokenDepositTx(params)` 🚧

Devnet only. See [Single-token deposit](single-token-deposit.md).

### `parseEventsFromLogs(logs): CubicPoolEvent[]`

Pure-function helper to extract Anchor events from a transaction's `meta.logMessages`. Useful for indexers / post-trade reporting.

```ts
const tx = await connection.getTransaction(sig, { commitment: "confirmed" });
const events = client.parseEventsFromLogs(tx?.meta?.logMessages ?? []);
for (const e of events) {
  if (e.kind === "Swap") {
    console.log("Swap:", e.amountIn, "→", e.amountOut);
  }
}
```

Decoded events: `Swap`, `LiquidityAdded`, `LiquidityRemoved`, `PoolStateLog`, `SwapFeeRateUpdated`, `ProtocolFeeRateUpdated`, `PoolEnabledUpdated`, `SwapsEnabledUpdated`, `MaxSelloffSet`, `MaxSelloffWindowAdvanced`, `RangeManagerSet`, `RangeManagerConfigSet`, `RangeManagerUpdated`, `ProtocolFeesCollected`.

### `helperPda(): PublicKey`

The per-pool helper PDA used by the single-token-deposit program. Mostly internal; exposed for advanced integrators that derive accounts manually.

---

## `CubeBackendClient` — REST wrapper

Wraps every endpoint in [API Reference](../integration/api-reference.md). Reads the API key from `cfg.apiKey`.

```ts
import { CubeBackendClient } from "@cubee_ee/sdk";
const backend = new CubeBackendClient({ config: cfg });
```

| Method | Endpoint |
|---|---|
| `listPools()` | `GET /api/pools` |
| `getPool(addr)` | `GET /api/pools/:address` |
| `listPoolsRaw(limit?, offset?)` | Pagination passthrough |
| `getPoolsByPair(a, b)` | `GET /api/pools/by-pair` |
| `getPoolsByAdmin(wallet)` | `GET /api/pools/my` |
| `getPortfolio(wallet?)` | `GET /api/pools/portfolio` |
| `getPlatformStats()` | `GET /api/pools/stats` |
| `listTokens()` | `GET /api/pools/tokens` |
| `getTopTokens(limit?)` | `GET /api/pools/top-tokens` |
| `getPoolTxs(addr, opts?)` | `GET /api/pools/:address/transactions` |
| `getPoolTxStats(addr)` | `GET /api/pools/:address/tx-stats` |
| `getSwapRoute(tokenIn, tokenOut, amountIn, decimalsIn?)` | `GET /api/pools/swap-route` |
| `getStats(metric, window?, pool?, unit?)` | `GET /api/stats/:metric` |
| `getLeaderboard(page, limit)` | `GET /api/leaderboard` |
| `getLeaderboardUser(addr)` | `GET /api/leaderboard/user/:address` |
| `getLeaderboardUserHistory(addr, from, to)` | `GET /api/leaderboard/user/:address/history` |
| `getLeaderboardEpoch()` | `GET /api/leaderboard/epoch` |
| `getTokenPrices(mints)` | Batched Pyth/Jup price feed |

All return `SdkResult<T>`. Each typed against the response shapes in [API Reference](../integration/api-reference.md).

```ts
const r = await backend.getSwapRoute(
  "So11111111111111111111111111111111111111112",
  "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "1000000000",
  9,
);
if (!r.ok) return console.error(r.error.code, r.error.humanMessage);
console.log(r.data.routes.length, "split-route legs");
```

---

## `PoolFactoryClient` — deploy new pools

Used by the create-pool flow on the frontend (and any external integrator that wants to spin up a pool).

```ts
import { PoolFactoryClient } from "@cubee_ee/sdk";
const factory = new PoolFactoryClient({ config: cfg });
```

### `buildInitializeConfigTx(params)`

Creates a `CubicPoolConfig` account (the per-config governance container). One config can host many pools.

```ts
const r = factory.buildInitializeConfigTx({
  payer: walletPubkey,
  defaultProtocolFeeRate: 2000,   // 20% of swap fees go to protocol
});
// returns { instructions, configKeypair: Keypair }
```

The config account is a fresh keypair (not a PDA) — it must sign the tx alongside the wallet.

### `buildDeployPoolTx(params)`

Creates a `CubicPool` under an existing config.

```ts
factory.buildDeployPoolTx({
  payer: walletPubkey,
  configKey: PublicKey,
  poolId: new BN(Date.now()),      // user-chosen salt → PDA seed
  tokens: [mintA, mintB, ...],     // 2–9 tokens (UI cap; 10 supported on-chain)
  weightsBps: [5000, 5000, ...],   // sum must = 10000
  virtualBalances: [new BN(...), ...],
  swapFeeRate: 3000,               // hundredths of bps; 3000 = 0.3%
});
// returns { instructions, pool, bptMint } — the latter two are derived PDAs the caller should remember
```

### `initializeCubicPoolIx(params)`

Low-level: just the bare `initialize_cubic_pool` instruction without ComputeBudget wrapping. Useful if you're batching pool creation with other ixs.

---

## `AdminClient` — admin / treasury ops

Used by the Admin Panel on the frontend and by ops scripts. Most methods build the **instruction** only — you compose into a tx + sign.

```ts
import { AdminClient } from "@cubee_ee/sdk";
const admin = new AdminClient({ config: cfg, provider });
```

| Group | Methods |
|---|---|
| Treasury | `initializeTreasuryIfMissing` (does check + send), `initiateAdminTransferIx`, `acceptAdminTransferIx`, `cancelAdminTransferIx` |
| Token registry | `registerTokenIx` |
| Withdrawals | `withdrawIx` (treasury) |
| Pool config | `poolInitializeConfigIx`, `setProtocolFeeRateIx`, `setPoolEnabledIx`, `setSwapsEnabledIx`, `setBannedExtensionsIx` |
| Fee collection | `collectProtocolFeesIx` |
| Emergency | `debugWithdrawLiquidityIx` |

For pool-admin ops that aren't yet wrapped by the SDK (`set_swap_fee_rate`, `set_range_manager`, `set_range_manager_config`, `set_max_selloff`, `initiate/cancel/accept_pool_admin_transfer`, `range_manager_update`), the [PoolAdmin UI](../safety/pool-controls.md) builds them directly via the IDL. PRs welcome to add SDK wrappers.

---

## `SingleTokenDepositClient` — devnet helper 🚧

Low-level wrapper around `buildSingleTokenDepositTx`. Not stable yet — see [Single-token deposit](single-token-deposit.md).

---

## Token-2022 support

Cube pools accept both classic SPL Token and Token-2022 mints, and a single pool can mix both programs. The SDK decodes `tokens[i].tokenProgram` from the on-chain pool account and threads it through every builder, so callers can pass mints straight from the pool — no extra plumbing required.

What this means in practice:

- `buildSwapTx`, `buildAddLiquidityTx`, `buildRemoveLiquidityTx`, and `buildSingleTokenDepositTx` derive user ATAs under the correct program and emit the right `token_program_i` in `remaining_accounts`.
- The BPT mint always uses classic SPL Token.
- Extensions that change transfer amounts (transfer fee, transfer hook, confidential transfer, non-transferable) are **banned** — the AMM math expects vault delta == requested amount, otherwise the contract reverts with `BannedExtension` during pool init.

---

## When NOT to use the SDK

- If you only need TVL / volume / fee data, hit the DefiLlama API (the protocol is listed under `cube`) — no key needed.
- If you only need pool addresses + token pairs and you're OK polling a REST endpoint, the backend's `/api/pools/by-pair` (with an API key) is enough.

For everything else (composing transactions, decoding events, streaming pool state with proper retries), use the SDK rather than re-deriving accounts from the IDL by hand.

---

## Versioning

The SDK follows semver. Breaking changes bump the minor version pre-1.0. Pin to a minor (`^0.2`) for stable behaviour; check the [CHANGELOG](https://github.com/cubee-ee/sdk/blob/main/CHANGELOG.md) before bumping.

Notable recent fixes:
- **`0.2.2`** — `buildSwapTx` no longer silently defaults `minAmountOut` to 0 when the internal quote fails (security: previously could build a tx accepting any output). Error map is now auto-generated from the contract IDL, so all error categorisations stay in sync with new Anchor errors.
