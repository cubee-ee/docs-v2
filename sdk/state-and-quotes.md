# Read pool state and calculate a swap

`CubicPoolClient.sync()` is the SDK operation that reads and returns the parsed
pool state. You do not need to reconstruct the selloff configuration from an
API response. It is present in `result.data.tokens[i]`, together with the
reserves and the window's runtime counters. This page maps every operational
pool field to that result and shows how those fields enter a quote.

## Choose the right read API

| API | Input | Result | Use it for |
| --- | --- | --- | --- |
| `await client.sync()` | Client configured with a pool address and RPC | `SdkResult<PoolInfo>` | Application quotes and UI: ordered tokens, decoded pool state, mint metadata, BPT supply and chain time |
| `client.getCached()` | No arguments | `PoolInfo \| undefined` | The last successful sync, without another RPC read |
| `decodePoolAccount(data)` | Current 1,683-byte pool `Buffer` | `RawPoolAccount` | RPC/indexer integration: camelCase pool fields and ten parallel per-token arrays |
| `decodeContractAccount("cubicPool", "CubicPool", data)` | Current pool `Buffer` | Typed IDL account | Exact nested `tokens[i].config` and `tokens[i].dynamics`, snake_case fields, all integer widths and reserved bytes |
| `decodeContractAccount("cubicPool", "CubicPoolConfig", data)` | Current 202-byte config `Buffer` | Typed IDL config | Protocol authority, pending authority, defaults and extension policy |
| `decodeContractAccount("protocolAdmin", "Treasury", data)` | Current 818-byte Treasury `Buffer` | Typed IDL Treasury | Protocol admin/supervisor and registered Treasury token vaults |

The decoders are synchronous and throw on invalid input. They do not fetch data
or prove account ownership: validate the RPC account's `owner` separately.
`sync()` checks pool and mint ownership, returns a structured error on a failed
read/decode, and replaces its cache only on success. If a later sync fails,
`getCached()` can still contain an older snapshot; do not treat that as fresh.
There is no `getState()` or `getSwapQuote()` method in this SDK. The actual
methods are `sync()`, `getCached()` and `quoteSwap()`.

`RawPoolAccount` keeps all ten token slots, including unused slots. `PoolInfo`
contains only the `tokenCount` active entries in pool order. Here “active entry”
means inside that prefix; its `isActive` input-trading flag may still be false.
The exact ABI decoder also preserves reserved bytes, whose contents have no
current operational meaning. Full offsets and serialized types are in
[Accounts and Events](../technical/accounts-events.md).

## Pool-level field mapping

Paths in the first column are fields of the decoded on-chain `CubicPool`.
The final column describes `PoolInfo`, returned by `sync()`. `PK` means
`PublicKey`; `BN` values must remain integers for arithmetic.

| Contract field | `RawPoolAccount` | `PoolInfo` type and meaning |
| --- | --- | --- |
| `config` | `config` | `config: PK`; config account address, not its decoded contents |
| `bump` | `bump` | `bump: number`; pool PDA bump |
| `pool_id` | `poolId` | `poolId: BN`; u64 PDA seed |
| `token_count` | `tokenCount` | `tokenCount: number`; number of meaningful token slots |
| `swap_fee_rate` | `swapFeeRate` | `swapFeeRate: number`; input fee, 1,000,000 = 100% |
| `protocol_fee_rate` | `protocolFeeRate` | `protocolFeeRate: number`; share of the input fee, 10,000 = 100% |
| `created_at` | `createdAt: BN` | `createdAt: number`; Unix seconds |
| `pool_enabled` | `poolEnabled` | `poolEnabled: boolean`; operational gate |
| `swaps_enabled` | `swapsEnabled` | `swapsEnabled: boolean`; trading gate |
| `pool_admin` | `poolAdmin` | `poolAdmin?: PK`; pool owner/administrator |
| `pending_pool_admin` | `pendingPoolAdmin` | `pendingPoolAdmin?: PK`; proposed successor |
| `range_manager` | `rangeManager` | `rangeManager?: PK`; delegated updater |
| `range_manager_enabled` | `rangeManagerEnabled` | `rangeManagerEnabled?: boolean` |
| `range_manager_max_vb_change_pct` | `rangeManagerMaxVbChangePct` | Same name, `number?`; per-call relative VB change bound, scale 10,000 |
| `range_manager_max_weight_change_pct` | `rangeManagerMaxWeightChangePct` | Same name, `number?`; per-call relative weight change bound, scale 10,000 |
| `range_manager_min_update_interval_secs` | `rangeManagerMinUpdateIntervalSecs` | Same name, `number?`; minimum seconds between updates |
| `range_manager_last_updated` | `rangeManagerLastUpdated` | Same name, `BN?`; signed Unix seconds |
| `range_manager_max_leverage_bps` | `rangeManagerMaxLeverageBps` | Same name, `number?`; maximum new VB/actual ratio, 10,000 = 1×; zero disables |
| `range_manager_min_leverage_bps` | `rangeManagerMinLeverageBps` | Same name, `number?`; minimum new VB/actual ratio, 10,000 = 1×; zero disables |
| `lookup_table` | `lookupTable` | `lookupTable: PK`; zero means no pool ALT |
| `banned_extensions` | `bannedExtensions` | `bannedExtensions: BN`; pool creation-policy bitmap |
| `tokens` | Parallel arrays in the next table | `tokens: PoolTokenInfo[]`; ordered token objects |
| `reserved` | `reserved` | No quote input; exact ABI decoding preserves these 16 bytes |

Optional fields in the TypeScript interfaces support older manually supplied
snapshots. A successful current `sync()` populates the operational fields above,
including zero values. Zero public keys mean an unset/renounced role according
to that field's contract semantics. They are not a missing RPC result.

## Every token field, including selloff and surge

For slot `i`, `config.*` below means `CubicPool.tokens[i].config.*` and
`dynamics.*` means `CubicPool.tokens[i].dynamics.*`. These are not the parent
`CubicPoolConfig` account.

| Contract slot field | Raw parallel array at `[i]` | `PoolInfo.tokens[i]` | Meaning in a swap |
| --- | --- | --- | --- |
| `config.mint` | `tokenMints` | `mint: PK` | Token identity |
| `config.token_program` | `tokenPrograms` | `tokenProgram: PK` | Token program used to derive ATAs and execute transfers |
| `config.normalized_weight` | `normalizedWeights: BN[]` | `weightBps: number` | Pricing exponent uses input weight / output weight; 10,000 = 100% |
| `config.is_active` | `isActive` | `isActive: boolean` | False rejects this token as swap input; does not by itself forbid receiving it |
| `config.max_selloff_pct` | `maxSelloffPct` | `maxSelloffPct: number` | Cap fraction of the window's VB snapshot, scale 10,000; zero disables limiter and surge |
| `config.max_selloff_period_length` | `maxSelloffPeriodLength` | `maxSelloffPeriodLength?: number` | Window period in seconds |
| `config.variable_fee_threshold_pct` | `variableFeeThresholdPct` | `variableFeeThresholdPct?: number` | Window fill at which output surge begins, scale 10,000 |
| `config.variable_fee_slope_low_pct` | `variableFeeSlopeLowPct` | `variableFeeSlopeLowPct?: number` | Fee rate at the threshold, scale 10,000; a control-point rate despite the word “slope” |
| `config.variable_fee_slope_mid_pct` | `variableFeeSlopeMidPct` | `variableFeeSlopeMidPct?: number` | Fee rate at the kink, scale 10,000 |
| `config.variable_fee_slope_high_pct` | `variableFeeSlopeHighPct` | `variableFeeSlopeHighPct?: number` | Fee rate at full fill, scale 10,000; zero disables surge, not the cap |
| `config.variable_fee_kink_pct` | `variableFeeKinkPct` | `variableFeeKinkPct?: number` | Kink fill in **whole percent**; 95 means 95%; zero selects a single line |
| `dynamics.virtual_balance` | `virtualBalances` | `virtualBalance: BN` | Raw pricing balance; also captures a zero/new window snapshot |
| `dynamics.actual_balance` | `actualBalances` | `actualBalance: BN` | Raw LP reserve; bounds gross output, already excludes protocol funds |
| `dynamics.protocol_fees_owed` | `protocolFeesOwed` | `protocolFeesOwed: BN` | Separate raw protocol balance; input base-fee share and output surge accrue here |
| `dynamics.previous_selloff` | `previousSelloff` | `previousSelloff?: BN` | Previous gross-input bucket; its contribution decays with elapsed time |
| `dynamics.current_selloff` | `currentSelloff` | `currentSelloff?: BN` | Current gross-input bucket; does not decay until rotation |
| `dynamics.window_start_timestamp` | `windowStartTimestamp` | `windowStartTimestamp?: BN` | Signed Unix seconds for current bucket start |
| `dynamics.selloff_vb_snapshot` | `selloffVbSnapshot` | `selloffVbSnapshot?: BN` | Raw VB snapshot fixing the cap inside the current window; zero requires live VB at projection |

All reserve/counter `BN`s use that slot's raw token units. A token with five
decimals has 100,000 raw units per displayed token. Do not apply mint decimals
to a percentage, and do not compare input-token selloff counters to an
output-token fee amount.

The gross amount offered to the pool consumes selloff capacity **before** the
base fee is removed. The input token supplies the entire window and fee curve,
but surge is retained in the **output** token. Parameters from the output token's
selloff configuration do not price this direction of the swap.

## Fields fetched or derived outside the pool account

| `PoolInfo` or token field | Source | Purpose |
| --- | --- | --- |
| `address` | Constructor's pool public key | Account identity |
| `bptMint` | PDA `["bpt_mint", pool]` | LP share mint |
| `bptTokenProgram` | BPT mint account owner | Correct BPT transfer/mint/burn program |
| `bptTotalSupply` | BPT mint data | LP join/exit math; the pool account does not store supply |
| `chainTimestamp` | Solana Clock `unix_timestamp` | Unix seconds for time-dependent quotes |
| `syncedAt` | Local `Date.now()` | Milliseconds for read-age display; **not** the quote's chain timestamp |
| `unsupportedTokenIndices` | Derived from each mint's extensions | Tokens the current SDK cannot safely transfer |
| `tokens[i].index` | Position in the meaningful prefix | Instruction token index |
| `tokens[i].decimals` | Token mint account data | Raw-unit conversion and display |
| `tokens[i].vault` | ATA(pool, mint, stored token program) | Vault address; sync does not read its current token balance |
| `tokens[i].metadata` | Config token registry or SDK known-token registry | Optional symbol/logo/display information; not contract state |
| `tokens[i].concentration` | `actualBalance / virtualBalance`, or zero for zero VB | Floating display ratio; inverse of VB/actual leverage; do not use it for integer quotes |
| `tokens[i].extensions` | Token-2022 mint TLV | Extension discriminants; empty for classic SPL Token |
| `tokens[i].unsupportedExtensions` | SDK runtime transfer rules | Operation compatibility; the supplied admission bitmap does not relax this list |

`sync()` first reads the pool, then batches mint/BPT/Clock reads. It does not
promise a single atomic multi-account slot, and it does not read config,
Treasury, vault token balances, helper dust, live market prices or route state.
For byte-exact i64/u64 values, use the raw/ABI decoder; `PoolInfo.createdAt` and
`chainTimestamp` use JavaScript numbers appropriate for ordinary Unix times.

## Read config and Treasury without losing fields

```typescript
import { Connection, PublicKey } from "@solana/web3.js";
import { decodeContractAccount, getConfig } from "@cubee_ee/sdk";

export async function readAuthorities(rpcUrl: string, configAddress: PublicKey) {
  const connection = new Connection(rpcUrl, "confirmed");
  const programs = getConfig("mainnet").programs;
  const [treasuryAddress] = PublicKey.findProgramAddressSync(
    [Buffer.from("treasury")], programs.protocolAdmin,
  );
  const [configInfo, treasuryInfo] = await connection.getMultipleAccountsInfo([
    configAddress, treasuryAddress,
  ]);
  if (!configInfo?.owner.equals(programs.cubicPool)) {
    throw new Error("Missing config or unexpected config owner");
  }
  if (!treasuryInfo?.owner.equals(programs.protocolAdmin)) {
    throw new Error("Missing Treasury or unexpected Treasury owner");
  }
  return {
    config: decodeContractAccount("cubicPool", "CubicPoolConfig", configInfo.data),
    treasury: decodeContractAccount("protocolAdmin", "Treasury", treasuryInfo.data),
  };
}
```

The config result contains `protocol_admin`, `pending_protocol_admin`,
`default_protocol_fee_rate`, `banned_extensions`, `hard_banned_extensions`, and
`reserved`. These are config defaults/authorities, not substitutions for a
pool's already stored fee rate or bitmap. Treasury returns `admin`,
`pending_admin`, `bump`, `token_count`, `token_mints`, `token_vaults`, `created_at`,
`reserved`, and `supervisor`. Use only the registered prefix of its ten-element
arrays. Treasury vault addresses are not pool ATAs. The current Treasury
decoder requires the 818-byte layout; a legacy 786-byte Treasury needs its
supervisor realloc before this decoder can read the full current schema.

## Read, inspect and quote

This example is a complete read-and-quote function. It does not need a signer.
The caller supplies a pool, token indices, gross input in raw units, and the
desired slippage budget. Use the refreshed quote's `minAmountOut` in the swap
builder; it already accounts for surge.

```typescript
import BN from "bn.js";
import { PublicKey } from "@solana/web3.js";
import { CubicPoolClient, getConfig } from "@cubee_ee/sdk";

export async function inspectAndQuote(
  rpcUrl: string,
  poolAddress: string,
  tokenInIndex: number,
  tokenOutIndex: number,
  amountInRaw: string,
  slippageHbps = 5_000, // 0.5%, not 50%
) {
  const client = new CubicPoolClient({
    config: getConfig("mainnet", { rpcEndpoints: [rpcUrl] }),
    poolAddress: new PublicKey(poolAddress),
  });
  const synced = await client.sync();
  if (!synced.ok) throw new Error(synced.error.humanMessage);
  const state = synced.data;
  const quote = client.quoteSwap(
    tokenInIndex, tokenOutIndex, new BN(amountInRaw), slippageHbps,
  );
  if (!quote.ok) throw new Error(quote.error.humanMessage);

  return {
    client,
    state,                         // every parsed operational field above
    input: state.tokens[tokenInIndex],
    output: state.tokens[tokenOutIndex],
    quote: quote.data,              // raw BN amounts, including the output floor
  };
}
```

`quoteSwap()` is synchronous after the read. It neither sends a transaction
nor advances the cached window. Repeating it gives independent alternatives
against the same state; it does not model several executed swaps in sequence.
For a single-token deposit, `quoteSingleTokenDeposit()` does advance a private
state copy between its internal legs. The low-level formulas and helpers for
independent computation are in [Max-Selloff Window](../for-traders/max-selloff.md)
and [Dynamic Fee](../for-traders/dynamic-fee.md).

## How the fields enter output and slippage

For gross input `A`, base rate `r`, and protocol share `p`, the quote composes:

```text
F       = ceil(A × r / 1,000,000)       input base fee
P       = ceil(F × p / 10,000)          protocol share of that same fee
A_curve = A − F
G       = weighted_curve(Vin, Vout, win, wout, A_curve)
S       = surge_output(window_before, window_after,
                       input_token_curve, A_curve, G, reserves, weights)
N       = G − S                        net output paid to the trader
minimum = floor(N × (1,000,000 − slippageHbps) / 1,000,000)
```

The window check uses `A`; the AMM curve uses `A_curve`. The protocol share `P`
is part of `F`, not another deduction from `A_curve`. `G` must fit the actual
output reserve before surge is withheld. The same trade can therefore fail an
actual-reserve limit, the gross-input selloff cap, or the signed net-output floor.

| `SwapQuote` field | Units and use |
| --- | --- |
| `amountIn` | Gross raw input `A` |
| `feeAmount` | Raw input base fee `F` |
| `protocolFeeAmount` | Raw input share `P`, already included in `F` |
| `grossAmountOut` | Raw output before surge `G`; populated by current quotes |
| `surgeFeeAmount` | Raw output protocol charge `S`; populated even when zero |
| `amountOut` | Raw output after surge `N`; do not subtract `S` again |
| `minAmountOut` | Signed raw-output floor calculated from `N` |
| `spotOut` | Base-fee-adjusted spot output without finite-trade curvature or surge |
| `priceImpactHbps` | `floor(max(0, spotOut − N) × 1,000,000 / spotOut)` for positive spot output; zero otherwise |

For example, if net quoted output is 990,000 raw units, 0.5% slippage gives
985,050 minimum raw output. Slippage is not an additional fee: execution pays
the then-calculated output if it meets that floor, otherwise the swap fails.
Other sales can fill the input window and raise surge between quote and execution,
reducing net output even when the ordinary price curve has moved little. A full
window fails `MaxSelloffExceeded`; increasing slippage cannot bypass that cap.

`priceImpactHbps / 10_000` is a displayed percentage. It includes finite-trade
curve impact and the output surge relative to the base-fee-adjusted spot quote.
The base fee and surge amounts have different mints and must be shown separately
or converted into a common value unit before adding them.

Use the cached `chainTimestamp` for a reproducible snapshot quote. The optional
fifth argument `nowSeconds` lets you model a specific timestamp with the same
reserves; it must be a nonnegative safe integer in seconds. It is not a prediction
of future reserves or a reservation of window capacity. Refresh the state when
the user signs, and never replace a quote error with a zero output floor.

## Keep the state complete

- Do not fetch only balances and omit `previousSelloff`, `currentSelloff`,
  `windowStartTimestamp` or `selloffVbSnapshot` for an enabled token. The SDK
  rejects a swap quote with missing required window state.
- Do not treat a zero `selloffVbSnapshot` as a disabled policy. Use live
  `virtualBalance` when projecting the first window or a rotated window.
- Do not derive the cap as a permanent percentage of the latest VB. Inside a
  window its snapshot remains fixed; LP operations rescale it explicitly.
- Do not cache pool-admin configuration indefinitely. `set_max_selloff` can
  replace parameters without clearing the existing volume counters.
- Do not subtract `protocolFeesOwed` from `actualBalance`; v5 already separates
  protocol-owned funds from LP reserves.
- Do not substitute an endpoint fee percentage for the full surge computation.
  It isolates the threshold and assigns output to four input subsegments.

The [SDK reference](reference.md) lists every convenience method and the generic
typed instruction builder. The [instruction reference](../technical/instruction-reference.md)
lists the complete 59-instruction ABI, including the methods with no dedicated
convenience wrapper.
