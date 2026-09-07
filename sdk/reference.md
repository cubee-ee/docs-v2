# SDK API reference

This reference describes the public API of `@cubee_ee/sdk` 0.11.1 at `27de819`. Contract argument and account names follow the shipped IDLs from contract revision `96a2ee2`. Backend method availability is checked separately in the [REST reference](../integration/api-reference.md).

In the tables below, `PK` means `PublicKey`, `BN` is `bn.js`, `Ix` means `TransactionInstruction`, and `Result<T>` means `SdkResult<T>`. An `async` result is `Promise<Result<T>>`. Optional parameters end in `?`; defaults are shown where relevant. These abbreviations describe signatures; they are not additional package exports.

## Configuration and common results

`getConfig(network, overrides?) → CubeConfig` accepts `"mainnet"` or `"devnet"`. Overrides are `rpcEndpoint?: string`, `rpcEndpoints?: string[]`, `rpcTimeoutMs?: number`, `rpcCommitment?: Commitment`, `cuLimit?: number`, `slippageHundredthsBps?: number`, `backendEndpoint?: string`, and `tokens?: Record<string, TokenInfo>`. See [configuration and units](index.md#installation-and-configuration).

`programId(config, kind) → PK` selects `"cubicPool"`, `"protocolAdmin"`, or `"singleTokenLiquidity"`. `NETWORK_PROGRAMS` exposes the same per-network addresses. `KNOWN_TOKENS` and `resolveKnownToken(mint: string) → TokenInfo | undefined` provide optional display metadata, not an on-chain mint allowlist.

`Result<T>` is `{ ok: true, data: T }` or `{ ok: false, error: SdkError }`. `SdkError` contains `code`, `humanMessage`, and optional diagnostic context. Use `ok(data)` and `err(code, message, cause?)` when composing result-based operations. `toSdkError(cause, program?)` maps exceptions, and `describeProgramError(code, program?)` looks up errors for `"cubicPool"`, `"protocolAdmin"`, or `"singleTokenLiquidity"`.

`BuiltTx` is `{ instructions: Ix[], extraSigners?: PK[], suggestedCuLimit: number }`. It does not contain a signed transaction. Some builders already include a compute-budget instruction; inspect the instruction set before adding another. Generic builders and low-level math/parsers can throw even where the high-level client returns `Result`.

## CubicPoolClient

Constructor: `new CubicPoolClient({ config, poolAddress, rpc? })`. `poolAddress` is a `PK`. `rpc` can be a `RpcClient` or endpoint options (`endpoint`, `endpoints`, `fallbackEndpoints`, `apiKey`, `commitment`, `timeoutMs`). Omit it to use the config's RPC defaults.

| Method | Arguments | Return and behavior |
| --- | --- | --- |
| `sync` | None | Async `PoolInfo`; reads current pool state, mint owners/extensions/decimals, BPT supply, and Clock; updates the cache on success |
| `getCached` | None | `PoolInfo \| undefined`; no I/O |
| `helperPda` | None | `PK` of the single-token helper |
| `quoteSwap` | `tokenInIndex: number, tokenOutIndex: number, amountIn: BN, slippageHundredthsBps?: number, nowSeconds?: number` | `Result<SwapQuote>` using cached state, input sell-off window, and dynamic output fee |
| `quoteSeedDeposit` | `user: PK, tokenAmounts: BN[], slippageHundredthsBps?: number` | `Result<AddLiquidityQuote>` for the initial admin deposit into a pool with zero BPT supply |
| `quoteAddLiquidity` | `tokenAmounts: BN[], slippageHundredthsBps?: number` | `Result<AddLiquidityQuote>` for a proportional join bounded by spend ceilings |
| `quoteRemove` | `bptIn: BN` | `Result<{ tokenOuts: BN[], effectiveBptIn: BN }>`; preserves 1,000 raw BPT and uses the contract's integer rounding |
| `quoteSingleTokenDeposit` | `tokenInIndex: number, amountIn: BN, slippageHundredthsBps?: number, nowSeconds?: number, helperBalances?: BN[]` | `Result<SingleTokenDepositQuote>`; simulates sequential swaps and proportional join |
| `buildSwapTx` | `SwapParams` below | `Result<BuiltTx>` |
| `buildAddLiquidityTx` | `AddLiquidityParams` below | `Result<BuiltTx>`; used for both seed and later joins |
| `buildRemoveLiquidityTx` | `RemoveLiquidityParams` below | `Result<BuiltTx>` |
| `buildSingleTokenDepositTx` | `SingleTokenDepositParams` below | `Result<BuiltTx>` with ATA setup and deposit together |
| `buildSingleTokenDepositTxs` | `SingleTokenDepositParams` | `Result<{ setup: BuiltTx \| null, deposit: BuiltTx }>` |
| `singleTokenDeposit` | Getter, not a call | `SingleTokenDepositClient` sharing this client |
| `parseEventsFromLogs` | `logs: string[]` | `CubicPoolEvent[]`, the camelCase compatibility event API |

Quotes require a successful `sync()`. Token indices and vectors use pool order. Amounts must fit the applicable unsigned integer widths. `nowSeconds` overrides cached Solana Clock time; it does not change the time an eventual transaction observes. The STLD `helperBalances` vector contains existing helper token ATA balances before the operation; omitted means zero, not “fetch automatically.”

`PoolInfo` includes pool/config/admin and pending-admin addresses, token count and ordered `tokens`, BPT mint/supply/token program, enabled/swap flags, fee rates, range-manager configuration, lookup-table address, extension policy, and sell-off/window state. Per-token fields include mint, vault, token program, decimals, weight, actual and virtual balances, protocol fees owed, activation, and extension information. Keep these values from the same logical sync. They are not an atomic multi-account snapshot.

### Quote results

| Type | Fields |
| --- | --- |
| `SwapQuote` | `tokenInIndex`, `tokenOutIndex`, `amountIn`, net `amountOut`, `grossAmountOut?`, `surgeFeeAmount?`, `spotOut`, `priceImpactHbps`, input `feeAmount`, input `protocolFeeAmount`, `minAmountOut` |
| `AddLiquidityQuote` | Input ceilings `tokenAmounts`, estimated credited `depositAmounts`, unspent `refundAmounts`, `bptOut`, slippage-adjusted `minimumBptAmount`, `limitingTokenIndex` (`-1` for seed) |
| `SingleTokenDepositQuote` | `tokenInIndex`, `amountIn`, `allocations`, `expectedOuts`, informational `minOuts`, actual-join estimate `depositedAmounts`, `refundAmounts`, `estimatedBpt`, `sidelinedTokenIndices` |

The monetary quantities in these results are `BN` values in the corresponding token's raw units. `priceImpactHbps` is a number in hundredths of a basis point. For swaps, gross output leaves pool reserves; the output-token surge fee reduces the user's net output. The input protocol fee is already separate from the LP reserve in v5 accounting.

Seed quotes require the pool's admin, an enabled pool, zero BPT supply, and a complete amount vector with at least one positive entry. The initial invariant uses the configured virtual basket unless all virtual balances are zero. It must produce at least 1,000 raw BPT. Later joins use the smallest spend-ceiling-to-actual-reserve ratio and reject unusably small deposits. See [mathematics](../technical/math.md) for the exact invariant, floors, and range checks.

### Transaction parameter objects

```ts
interface SwapParams {
  user: PublicKey;
  tokenInIndex: number;
  tokenOutIndex: number;
  amountIn: BN;
  slippageHundredthsBps?: number;
  minAmountOut?: BN;
}
interface AddLiquidityParams {
  user: PublicKey;
  tokenAmounts: BN[];
  minimumBptAmount?: BN;
}
interface RemoveLiquidityParams {
  user: PublicKey;
  bptAmount: BN;
  minimumTokenAmounts?: BN[];
}
interface SingleTokenDepositParams {
  user: PublicKey;
  tokenInIndex: number;
  amountIn: BN;
  slippageHundredthsBps?: number;
  minimumBptAmount?: BN;
}
```

Although retained as optional in the TypeScript interfaces, `minimumBptAmount` is **required and positive at runtime** for add/STLD builders, and `minimumTokenAmounts` is required for remove builders. An explicit zero entry in the remove vector is allowed. Supply all entries in pool order. `buildSwapTx` uses an explicit `minAmountOut` if given; otherwise it computes a quote and propagates a quote failure instead of silently using zero.

## SingleTokenDepositClient

Constructor: `new SingleTokenDepositClient({ config, poolAddress, rpc?, poolClient? })`. `poolClient` can be an existing `CubicPoolClient` for shared state; `rpc` uses the same options as above.

| Method | Signature and return |
| --- | --- |
| `helperPda` | `() → PK` |
| `sync` | `() → Promise<Result<PoolInfo>>` |
| `quote` | `(tokenInIndex, amountIn, slippageHundredthsBps?, nowSeconds?, helperBalances?) → Result<SingleTokenDepositQuote>`; identical argument types to the parent client's quote |
| `buildTx` | `(params: SingleTokenDepositParams) → Result<BuiltTx>` |
| `buildTxs` | `(params: SingleTokenDepositParams) → Result<{ setup: BuiltTx \| null, deposit: BuiltTx }>` |

See [single-token deposits](single-token-deposit.md) for helper balances, sequential window updates, ATA setup, final BPT floors, and event interpretation. Its quote checks affected token legs, but all STLD instruction builders reject incompatible extensions in **any** pool mint, including a sidelined mint, because helper refunds may touch it.

## PoolFactoryClient

Constructor: `new PoolFactoryClient({ config })`.

| Method | Arguments | Return |
| --- | --- | --- |
| `buildDeployPoolTx` | `DeployPoolParams` | `Result<BuiltTx & { pool: PK, bptMint: PK }>` |
| `buildInitializeConfigTx` | `{ payer: PK, defaultProtocolFeeRate: number }` | `Result<BuiltTx & { configKeypair: Keypair }>` |
| `initializeCubicPoolIx` | `DeployPoolParams` | `Ix`; can throw |

`DeployPoolParams` is `{ payer: PK, configKey: PK, poolId: BN, tokens: PK[], weightsBps: number[], virtualBalances: BN[], swapFeeRate: number, bptTokenProgram?: PK, bannedExtensions?: BN | number | null }`. Supply two to ten tokens and equal-length arrays, weights summing to `10_000`, and positive virtual balances. `poolId` is the u64 ID used in PDA derivation. `payer` becomes the creator/admin under initialization rules and funds new accounts. The mint list is encoded as remaining accounts, not an instruction-data token vector.

Omitting `bptTokenProgram` selects classic SPL Token. Omitting or passing null for `bannedExtensions` inherits the config default; an explicit bitmap selects the pool's configurable policy, still subject to the config hard floor and runtime transfer restrictions. Read existing policy accounts; do not assume all configs contain fresh-initialization defaults.

For config initialization, `payer` must be `Treasury.admin`. Sign with both that admin and the returned `configKeypair`. The factory routes this operation through `protocolAdmin.pool_initialize_config`, because the underlying config initializer needs the Treasury PDA's CPI signature.

## Instruction and transaction builders

All the following are named package exports. They build locally and can throw. They do not fetch state, sign, or submit. `cfg` is `CubeConfig`; `pool` is `PoolInfo` in the first table. An `Ix` function returns a raw instruction; a `Tx` function returns `BuiltTx` unless stated otherwise.

| Exports | Parameters |
| --- | --- |
| `buildSwapIx`, `buildSwapTx` | `(cfg, pool, params: SwapParams & { minAmountOut: BN })` |
| `buildAddLiquidityIx`, `buildAddLiquidityTx` | `(cfg, pool, params: AddLiquidityParams)` |
| `buildRemoveLiquidityIx`, `buildRemoveLiquidityTx` | `(cfg, pool, params: RemoveLiquidityParams)` |
| `buildSingleTokenDepositIx`, `buildSingleTokenDepositTx` | `(cfg, pool, params: SingleTokenDepositParams)` |
| `buildSingleTokenDepositTxs` | Same parameters; returns `{ setup: BuiltTx \| null, deposit: BuiltTx }` |
| `buildSingleTokenDepositAtaIxs` | `(cfg, pool, user: PK) → Ix[]`; idempotent token-account setup |
| `buildInitializeConfigIx` | `(cfg, { config: PK, payer: PK, defaultProtocolFeeRate: number })`; raw pool-program initializer with its CPI-only authority constraints |
| `buildPoolInitializeConfigIx` | `(cfg, { config: PK, admin: PK, defaultProtocolFeeRate: number })`; protocol-admin wrapper |
| `buildInitializeCubicPoolIx`, `buildDeployPoolTx` | `(cfg, params: DeployPoolParams)` |
| `buildInitializePoolAltIx` | `(cfg, params: InitializePoolAltParams) → Ix` |
| `buildInitializePoolAltTx` | Same parameters; returns `BuiltTx & { lookupTable: PK }` |
| `deriveAltAddress` | `(authority: PK, recentSlot: BN) → PK` |

`InitializePoolAltParams` is `{ pool: PK, config: PK, authority: PK, payer: PK, recentSlot: BN }`. ALT initialization is authorized by the pool admin or protocol-admin CPI path. Creation, extension, and freezing are part of the contract flow. The table is not available to a dependent transaction in the slot in which its addresses were added.

These management builders take a **pool address** (`PK`), not `PoolInfo`:

| Export | Parameters after `cfg` | Purpose |
| --- | --- | --- |
| `buildSetSwapFeeRateIx` | `pool, authority: PK, swapFeeRate: number` | Update the pool-admin swap fee |
| `buildSetMaxSelloffIx` | `pool, authority: PK, params: SelloffParams[]` | Replace the complete per-token sell-off policy vector |
| `buildSetRangeManagerIx` | `pool, params: SetRangeManagerParams` | Assign/enable a manager, subject to authority restrictions |
| `buildSetRangeManagerConfigIx` | `pool, params: SetRangeManagerConfigParams` | Set rate, interval, and leverage bounds |
| `buildRangeManagerUpdateIx` | `pool, params: RangeManagerUpdateParams` | Apply sparse compare-and-swap parameter changes |
| `buildInitiatePoolAdminTransferIx` | `pool, authority: PK, newAdmin: PK` | Nominate a successor |
| `buildAcceptPoolAdminTransferIx` | `pool, newAdmin: PK` | Accept as the nominee |
| `buildCancelPoolAdminTransferIx` | `pool, authority: PK` | Cancel a pending transfer |
| `buildDisablePoolAdminIx` | `pool, authority: PK` | Disable the pool-admin role under contract rules |
| `buildGetPoolInfoIx` | `pool` | Build the state-logging instruction; this is not an RPC account read |

All rows return `Ix`. The argument structures are:

```ts
interface SelloffParams {
  maxSelloffPct: number;
  periodLength: number;
  feeThresholdPct: number;
  feeSlopeLowPct: number;
  feeSlopeHighPct: number;
  feeSlopeMidPct: number;
  feeKinkPct: number;
}
interface SetRangeManagerParams {
  config: PublicKey;
  authority: PublicKey;
  newManager: PublicKey;
  enabled: boolean;
}
interface SetRangeManagerConfigParams {
  authority: PublicKey;
  maxVbChangePct: number;
  maxWeightChangePct: number;
  minUpdateIntervalSecs: number;
  maxLeverageBps: number;
  minLeverageBps: number;
}
interface TokenChange {
  index: number;
  expectedCurrent: BN;
  newValue: BN;
}
interface RangeManagerUpdateParams {
  authority: PublicKey;
  vbChanges?: TokenChange[];
  weightChanges?: TokenChange[];
}
```

`TokenChange` serializes `index`, `expected_current`, then `new_value`. Read `expectedCurrent` and compute `newValue` from the same state; stale updates fail atomically. `SetRangeManagerParams.config` is a required account. Percent fields use `10_000 = 100%`, except `feeKinkPct` (whole percent); period and interval fields use seconds. A zero leverage bound disables that bound. See [pool controls](../safety/pool-controls.md) for who may perform each operation.

### Versioned transactions

`buildVersionedTx(conn: Connection, payer: PK, instructions: Ix[], lookupTable: PK | undefined) → Promise<Result<{ tx: VersionedTransaction, alts: AddressLookupTableAccount[] }>>` fetches an optional ALT and a recent confirmed blockhash and compiles an unsigned v0 message.

`compileBuiltTx(conn, payer, built: BuiltTx, pool: Pick<PoolInfo, "lookupTable">)` returns the same shape, using the pool's ALT. Neither helper sends, signs, creates ATAs, initializes an ALT, or confirms execution. An ALT fetch failure returns a result error; blockhash and compilation exceptions can propagate.

## AdminClient

Constructor: `new AdminClient({ config, provider: AnchorProvider })`. The client exposes its Anchor `program` and `treasuryPda`. All methods ending in `Ix` return **`Promise<Ix>`**, without sending. The provider does not turn an arbitrary wallet into a valid admin: the current on-chain authority must sign.

In this table, unspecified addresses are `PK`; amounts and slots are `BN`. `tokenProgram?` defaults to classic SPL Token. `tokenPrograms?` defaults to a classic-program array matching `vaults`; pass actual mint programs for mixed pools.

| Method | Arguments | Operation |
| --- | --- | --- |
| `programDataPda` | None | Returns the protocol-admin ProgramData `PK` synchronously |
| `initializeTreasuryIfMissing` | `connection: Connection, admin, payer = admin` | Returns `Promise<boolean>`; checks existence and **submits initialization** if missing |
| `initializeTreasuryIx` | `payer, admin` | Build Treasury initialization |
| `initiateAdminTransferIx` | `admin, newAdmin` | Nominate a new Treasury admin |
| `acceptAdminTransferIx` | `newAdmin` | Accept the Treasury role |
| `cancelAdminTransferIx` | `admin` | Cancel the pending Treasury transfer |
| `setSupervisorIx` | `admin, newSupervisor` | Update the supervisor |
| `registerTokenIx` | `admin, mint, tokenProgram?` | Register a Treasury token account; this deployed Treasury path uses classic SPL Token |
| `withdrawIx` | `admin, mint, recipient, amount, tokenProgram?` | Withdraw Treasury tokens under the token constraints |
| `withdrawSolIx` | `admin, recipient, amount` | Withdraw available Treasury SOL |
| `upgradePoolProgramIx` | `admin, program, buffer, spill` | Upgrade a loader program through Treasury authority; buffer preparation is separate |
| `transferUpgradeAuthorityIx` | `admin, program, newAuthority` | Transfer loader authority; new authority is a required signer in this ABI |
| `freezePoolProgramIx` | `admin, program` | Revoke loader upgrade authority under the contract's freeze operation |
| `closePoolProgramIx` | `admin, program, recipient` | Close an authorized loader program |
| `poolInitializeConfigIx` | `admin, config, defaultProtocolFeeRate: number` | Initialize pool governance config through Treasury CPI |
| `poolInitiateProtocolAdminTransferIx` | `admin, config, newAdmin` | Start config protocol-admin transfer |
| `poolAcceptProtocolAdminTransferIx` | `admin, config` | Accept config protocol-admin transfer through Treasury |
| `poolCancelProtocolAdminTransferIx` | `admin, config` | Cancel that pending transfer |
| `setProtocolFeeRateIx` | `admin, config, pool, protocolFeeRate: number` | Set the protocol's share of the pool swap fee |
| `setPoolEnabledIx` | `admin, config, pool, enabled: boolean` | Change pool-enabled state |
| `setSwapsEnabledIx` | `admin, config, pool, enabled: boolean` | Change swap-enabled state |
| `setBannedExtensionsIx` | `admin, config, banned: BN, hardBanned: BN` | Set config default and hard-floor bitmaps |
| `migratePoolToV5Ix` | `admin, config, pool, reactivateTokens = false` | Migrate supported older pool layout/state |
| `freezePoolsIx` | `authority, pairs: { config: PK, pool: PK }[]` | Batch freeze; Treasury admin or supervisor |
| `unfreezePoolsIx` | `admin, pairs: { config: PK, pool: PK }[]` | Batch unfreeze; current contract also authorizes the supervisor |
| `setTokenActiveIx` | `authority, config, pool, tokenIndex: number, isActive: boolean` | Set input activation; current contract authorizes admin/supervisor for either value |
| `poolInitializeAltIx` | `admin, config, pool, recentSlot: BN` | Initialize pool ALT via Treasury CPI |
| `collectProtocolFeesIx` | `admin, config, pool, vaults: PK[], recipients: PK[], tokenPrograms?: PK[]` | Collect accrued fees; arrays in pool order |
| `debugWithdrawLiquidityIx` | `admin, config, pool, amounts: BN[], vaults: PK[], recipients: PK[], tokenPrograms?: PK[]` | Administrative liquidity withdrawal exposed by this ABI |
| `poolWithdrawSolIx` | `admin, config, source, recipient, amount` | Authorized pool-program SOL withdrawal |
| `stldWithdrawSolIx` | `admin, config, pool, source, recipient, amount` | Authorized single-token-helper SOL withdrawal |

These methods construct authority-sensitive instructions, not a deployment or signing workflow. They do not upload buffers, pair a phone, or perform Ledger signing. A Treasury PDA signs only inside its owning program's CPI; use the corresponding wrapper instead of expecting a wallet to sign for that PDA. Account relationships, rent reserves, and authority constraints remain enforced by the contracts.

## Complete typed contract ABI

Use the generic API for every instruction, account, and event, including operations without a convenience method. `ContractProgram` is `"cubicPool" | "protocolAdmin" | "singleTokenLiquidity"`.

```ts
buildContractInstruction(
  config,
  program,
  instruction,
  args,
  accounts,
  remainingAccounts?,
): TransactionInstruction
```

The type parameters derive `args` and `accounts` from `ContractInstructionMap[program][instruction]`. Names use exact **IDL snake_case**, including nested structs. All fixed accounts are explicit, including program IDs and sysvars; there is no automatic Anchor account resolution. Fixed signer/writable flags and order come from the IDL. Supply ordered `AccountMeta[]` for remaining accounts; their semantic layout is described in the [instruction reference](../technical/instruction-reference.md).

The builder checks missing/unknown fields, primitive types, integer widths, and fixed array lengths. Values wider than 32 bits use `BN`; smaller integers use numbers; absent options use `null`, not `undefined`. It does not validate every on-chain relationship or satisfy required signatures.

```ts
import { buildContractInstruction } from "@cubee_ee/sdk";

const ix = buildContractInstruction(
  config,
  "cubicPool",
  "get_pool_info",
  {},
  { pool: poolAddress },
);
```

The example's `poolAddress` is a `PublicKey`. It only constructs the instruction.

| Decoder | Arguments | Return |
| --- | --- | --- |
| `decodeContractAccount` | `(program, accountName, data: Buffer)` | Corresponding `ContractAccountMap` type; validates current size and discriminator; throws on mismatch |
| `decodeContractEvent` | `(base64: string, program?)` | `ContractEvent \| null` for the payload after `Program data:` |
| `parseContractEvents` | `(logs: string[], program?)` | All recognized complete `ContractEvent[]` entries |
| `decodePoolAccount` | `(data: Buffer)` | `RawPoolAccount`; dedicated pool layout decoder |
| `decodeMintAccount` | `(data: Buffer)` | `RawMintAccount`; supply, decimals, authorities, initialization and extension data |
| `parseCubicPoolEvents` | `(logs: string[])` | Compatibility `CubicPoolEvent[]`; normalizes known events and falls back to the current IDL decoder for other events |

`ContractEvent` is a discriminated union `{ program, kind, data }`; `data` retains IDL field names and all current event fields. The complete parser covers 60 known events across the three IDLs. Decoding is not proof of account ownership, transaction success, or the emitting program's identity. Check RPC ownership and confirmed execution/provenance before using decoded data for indexing.

`BorshReader(buffer)` provides `remaining(): number`, `skip(n): void`, `u8/u16/u32(): number`, `u64/i64(): BN`, `bool(): boolean`, `pubkey(): PK`, `vecU64(): BN[]`, and `vecPubkey(): PK[]`. It is a low-level cursor, not a substitute for ABI and owner validation.

## RPC fallback

Constructor: `new RpcClient({ endpoint?, endpoints?, fallbackEndpoints?, apiKey?, commitment?, headers?, timeoutMs?, backoffMs?, connectionFactory? })`. Endpoints are strings, headers are a string map, times are milliseconds, and `connectionFactory` is a test injection hook. A nonempty `endpoints` array wins over `endpoint` and `fallbackEndpoints`. The `apiKey` option appends an `api-key` query value to a single primary endpoint; with an explicit endpoint list, configure authentication in each URL or through headers.

| Method/property | Return |
| --- | --- |
| `endpoints` | Readonly property exposing the configured `string[]` |
| `connection` | Raw `Connection` for the currently active endpoint; direct calls bypass wrapper retry behavior |
| `getAccountInfo(pk, retry?)` | Async `{ data: Buffer, owner: PK, lamports: number } \| null` |
| `getMultipleAccountsInfo(pks: PK[], retry?)` | Async `(Buffer \| null)[]` |
| `getMultipleAccountsWithInfo(pks: PK[], retry?)` | Async `({ data, owner, lamports } \| null)[]` |
| `getSlot(retry?)` | Async `number` |
| `call<T>(fn: (connection: Connection) => Promise<T>, retry?)` | Async `T`; use for suitable repeatable RPC reads |

“Async” above means `Promise<Result<...>>`. The wrapper starts at the last successful endpoint and rotates on retryable timeout/unavailable/rate-limit errors. By default it attempts each configured endpoint once, with a two-second timeout per attempt and no backoff. A successful `null` account response does not trigger another endpoint. `retry` supports `attempts`, `timeoutMs`, and `shouldRetry`; in this revision, backoff is taken from the constructor, even though the shared `RetryOptions` type also contains `backoffMs`.

`safeCall(operation, options?) → Promise<Result<T>>` is the separate general retry wrapper. Its defaults are three attempts, 15-second timeouts, and `[200, 500, 1500]` millisecond backoff. It honors `RetryOptions.backoffMs`. Neither wrapper provides transaction idempotency or safe automatic re-signing.

## CubeBackendClient

Constructor: `new CubeBackendClient({ apiEndpoint, apiKey?, defaultHeaders?, onTokenRefreshed?, onAuthExpired? })`. `apiEndpoint` is the base URL without an endpoint suffix. `apiKey`, if supplied, initializes a **Bearer Authorization token**; it is not an `X-Cube-Api-Key` header. `defaultHeaders` is a string map. Callback signatures are `(tokens: AuthTokens) => void` and `() => void`.

All HTTP methods return `Promise<Result<...>>`. Named response types below are exported. Generic `<T>` is a compile-time assertion, not runtime response validation. The [REST reference](../integration/api-reference.md) identifies which methods have corresponding routes in the checked backend source and where responses differ.

### Discovery, routing, and analytics

| Method and arguments | Declared result data |
| --- | --- |
| `listPools()` | `PoolSummary[]`; see the envelope caveat below |
| `getPool(addr: string)` | `PoolSummary`; see the envelope caveat below |
| `listPoolsRaw(limit: number, offset: number)` | `{ data: unknown[], hasMore: boolean, totalCount: number }` |
| `getPoolRaw(addr: string)` | Unwrapped `unknown` pool |
| `createPool<T>(body: unknown)` | `T`; POST metadata to `/api/pools`, not a contract deployment |
| `getPoolsByTokenPair<T>(tokenA: string, tokenB: string)` | `T` |
| `getPlatformStats()` | `PlatformStatsResponse`: total actual/virtual TVL, 24h volume, pool count, update time |
| `getPortfolio<T>(wallet: string)` | `T`; public pool/BPT lookup, distinct from authenticated portfolio analytics |
| `getAllTokens<T>()` | `T` |
| `getTopTokens<T>(limit = 20)` | `T` |
| `getPoolTxStats<T>(addr: string)` | `T` |
| `getTransactions<T>(addr: string, options?: { limit?: number, offset?: number, type?: "swap" \| "add_liquidity" \| "remove_liquidity", user?: string })` | `T` |
| `getSwapRoute(tokenIn: string, tokenOut: string, amountIn: string, decimalsIn = 9, slippageBps?: number, pool?: string)` | `SwapRouteResponse`; raw integer amounts are decimal strings; verify deployed route support and response shape |
| `getTokenPrices(mints: string[])` | `PriceMap`: mint-address-to-USD-price map |
| `getStats(kind: StatsKind, window: StatsWindow = "7d", poolAddr?: string, unit: "usd" \| "token" = "usd")` | `StatsSeries`: `{ points: { t: number, v: number }[] }` |
| `getTokenPairChart(a: string, b: string, range: PairChartRange)` | `TokenPairChartResponse`: current A/B ratio and USD prices, change, granularity, `[unixSeconds, ratio][]` |

`StatsKind` is `tvl`, `volume`, `swap_count`, `avg_swap`, `median_swap`, `fees_lp`, `fees_protocol`, `users_total`, `dau`, `mau`, `deposits`, or `removals`. `StatsWindow` is `1d`, `7d`, `30d`, or `all`; the checked backend interprets `all` as 365 days and ignores `unit`. `PairChartRange` is `1d`, `1w`, `1m`, `1y`, or `all`.

`SwapRouteResponse` declares `routes`, `totalAmountIn`, `totalExpectedOut`, `minReceived`, `slippageBps`, `effectivePrice`, `priceImpact`, `spotPrice`, and `estimatedXp`. Each declared route has pool address/name, `amountIn`, `expectedOut`, `minAmountOut`, allocation percentage, fee, token programs, indices, and nullable vault addresses. The checked backend does **not** emit all these fields; do not infer their presence from TypeScript. See [routing](../integration/swap-routing.md).

`listPools()` and `getPool()` do not unwrap every backend envelope even though their declared types suggest a direct data object. For the checked backend, prefer `listPoolsRaw()` and `getPoolRaw()`, or validate the actual raw JSON yourself.

### Authentication and pool metadata administration

| Method and arguments | Declared result data |
| --- | --- |
| `getNonce(wallet: string)` | `NonceResponse`: `{ nonce, message }` |
| `verifySignature(message: string, signature: string)` | `AuthTokens`: `{ accessToken, refreshToken, wallet, expiresIn }` |
| `getTxChallenge(wallet: string)` | `TxChallengeResponse`: nonce, base64 unsigned transaction, memo; route availability must be checked |
| `verifyTransaction(signedTransactionBase64: string)` | `AuthTokens`; route availability must be checked |
| `setTokens(accessToken: string, refreshToken: string)` | `void`; stores both in memory |
| `clearTokens()` | `void`; removes both |
| `setAccessToken(token: string)` | `void`; deprecated access-only setter |
| `clearAccessToken()` | `void`; deprecated access-only clearer |
| `getAdminPools()` | `AdminPoolsResponse`: `pools` entries with pool metadata and admin information |
| `isPoolAdmin(poolAddress: string)` | `IsAdminResponse`: eligibility and pool address |
| `renamePool(poolAddress: string, name: string)` | `RenamePoolResponse`: result with updated pool metadata |
| `updatePoolSettings<T>(poolAddress: string, settings: { baseAssetMint?: string, description?: string })` | `T`; off-chain metadata only; check route availability |

On a non-auth request's 401, a stored refresh token enables one shared refresh request to `/api/auth/refresh` and one retry of the original request. `onTokenRefreshed` lets your application persist new tokens. Failed refresh clears stored credentials and calls `onAuthExpired`. Storage persistence and wallet signing are the application's responsibility.

### XP, referrals, and campaigns

| Method and arguments | Declared result data |
| --- | --- |
| `getLeaderboard(page = 1, limit = 20)` | `LeaderboardResponse`: `total`, `page`, `limit`, ranked `data` |
| `getLeaderboardUser(address: string)` | `LeaderboardUserStats`: rank, points, last accrual data/time |
| `getLeaderboardUserHistory(address: string, page = 1, limit = 50)` | `XpAccrualHistoryResponse`: paginated accrual rows |
| `getLeaderboardEpoch()` | `LeaderboardEpochResponse`: epoch bounds, multiplier, base/current rates, epoch schedule |
| `getLeaderboardStats()` | `LeaderboardStatsResponse`: total users and XP |
| `bindReferral(code: string, utm?: { source?: string, medium?: string, campaign?: string, content?: string, term?: string })` | `ReferralBindResponse` |
| `getReferralStatus()` | `ReferralStatusResponse`: referrer, codes, rates, and aggregate stats |
| `getMyReferrals(page = 1, limit = 20)` | `ReferralListResponse`: paginated referrals |
| `joinCampaign()` | `CampaignStatusResponse`: campaign, participation, join time |
| `getCampaignStatus()` | `CampaignStatusResponse` |
| `getCampaignInfo()` | `CampaignInfoResponse`: campaign dates, prize tiers, rates, status |
| `getCampaignRank(from?: string, to?: string)` | `CampaignRankResponse`: personal interval rank and totals |
| `getCampaignTop(from?: string, to?: string, page = 1, limit = 20)` | `CampaignTopResponse`: paginated interval ranking |

Campaign methods describe an SDK-side HTTP interface; their routes are absent from the checked backend revision. They do not establish an active campaign or entitlement to a prize. XP and referral behavior verified in backend source is documented in [Cube XP](../rewards/cube-xp.md), including legacy field names whose meaning changed.

### Authenticated portfolio analytics

| Method and arguments | Declared result data |
| --- | --- |
| `getPortfolioSummary()` | `PortfolioSummaryResponse`: wallet/LP values, changes, and aggregate portfolio metrics |
| `getPortfolioExposure()` | `PortfolioExposureResponse`: look-through token exposures |
| `getPortfolioHistory(range: "7d" \| "30d" \| "90d" \| "all" = "30d")` | `PortfolioHistoryResponse`: historical portfolio points |
| `getPortfolioPools()` | `PortfolioPoolsResponse`: portfolio pool entries |
| `getPortfolioPoolHistory(poolAddress: string, range = "30d")` | `PortfolioPoolHistoryResponse`; same range union as above |
| `getPortfolioChart(metric: PortfolioChartMetric, range: PortfolioChartRange)` | `PortfolioChartResponse`: metric-specific chart and summary data |
| `getPortfolioActivity(options?)` | `PortfolioActivityResponse`: `total/page/limit/data` activity rows |
| `getPortfolioHoldings(options?)` | `PortfolioHoldingsResponse`: `total/page/limit/data` token holdings with wallet/pool source breakdown |
| `getPortfolioPositions(options?)` | `PortfolioPositionsResponse`: `total/page/limit/totalValueUsd/data` position metrics |

For the last three methods, `options` contains optional numeric `page` and `limit`, `order: "asc" | "desc"`, and `sort`. Activity `sort` is `"time" | "value"`; it also accepts `type: "all" | "liquidity" | "zap" | "swap" | "transfer" | "deployed"`. Holdings `sort` is `"value" | "price" | "balance"`. Positions `sort` is `"value" | "fees" | "il" | "pnl" | "apy"`. Chart metrics are `"networth" | "pnl" | "il" | "xp"`; chart ranges are `"24h" | "1w" | "1m" | "1y" | "all"`.

Chart/activity/holdings/positions methods have no corresponding route in the checked backend source. Their declared fields are interface expectations, not evidence that a live server computes them. Other portfolio endpoints also depend on indexed data and valuation inputs; no SDK return type guarantees accuracy or freshness.

### Raw HTTP methods

`get<T>(path)`, `post<T>(path, body: unknown)`, and `put<T>(path, body: unknown)` return `Promise<Result<T>>` for a path relative to `apiEndpoint`. They preserve raw JSON structure and use the client's headers and auth handling. Use them only with a verified endpoint contract; generic `T` does not parse or validate its contents.

## Address, extension, and mathematical utilities

| Address helper | Return |
| --- | --- |
| `derivePoolPda(programId: PK, config: PK, poolId: BN)` | `[PK, bump: number]` |
| `deriveBptMint(cubicPoolProgramId: PK, pool: PK)` | `[PK, number]` |
| `deriveHelperPda(stldProgramId: PK, pool: PK)` | `[PK, number]` |
| `deriveTreasuryPda(protocolFeesProgramId: PK)` | `[PK, number]` |
| `deriveAta(owner: PK, mint: PK, tokenProgram: PK)` | `PK` |

`MintExtension` names the known extension type IDs. `parseMintExtensions(data: Buffer | Uint8Array) → number[]` parses a mint's extension list and can reject malformed data. `mintExtensionName(type: number) → string` formats a type. `unsupportedMintExtensions(extensions: number[], bannedExtensions?: BN | bigint) → number[]` checks runtime incompatibility; the second argument is retained for compatibility and does not relax the runtime list. `bannedMintExtensions(extensions, bannedExtensions?) → number[]` checks the selected bitmap. `describeUnsupportedToken(token) → string` formats the token's incompatibility, and `assertTokensSupported(pool, indices: number[] | "all", op: string) → void` throws if a requested leg is unsupported. `HARD_UNSUPPORTED_MINT_EXTENSIONS` is an SDK runtime list, not the on-chain config's hard-ban bitmap.

The math modules export fixed-point multiplication/division and bounds helpers, logarithm/exponent/power, weighted invariant, swap/BPT/spot calculations, proportional allocation, slippage and fee rounding, sell-off window advancement/rescaling, and dynamic fee integration. Their formulas and numerical conventions are in [pool mathematics](../technical/math.md); the exported signatures are listed below. In particular, `computeTwoTokenOptimalAllocations` is an analytical helper, not the allocation rule used by the deployed STLD program. For executable trade estimates use the high-level quotes, which compose all state-dependent checks.

### Exported math signatures

These functions are pure and return values directly; invalid domains, overflow, and applicable state limits can throw. They operate on raw integer amounts or `ONE = 10^18` fixed-point values as specified in [mathematics](../technical/math.md). No individual primitive is a complete swap, join, or deposit quote.

```ts
export declare function assertU64(value: bigint, label?: string): bigint;
export declare function assertU128(value: bigint, label?: string): bigint;
export declare function assertI128(value: bigint, label?: string): bigint;
export declare function assertInteger(value: number, min: number, max: number, label: string): number;
export declare function mulDivDown(a: bigint, b: bigint, denominator: bigint): bigint;
export declare function mulDown(a: bigint, b: bigint): bigint;
export declare function mulUp(a: bigint, b: bigint): bigint;
export declare function divDown(a: bigint, b: bigint): bigint;
export declare function divUp(a: bigint, b: bigint): bigint;
export declare function complement(x: bigint): bigint;
export declare function weightToFp(weightBps: bigint | number): bigint;
export declare function lnFp(a: bigint): bigint;
export declare function expFp(x: bigint): bigint;
export declare function powFp(base: bigint, exponent: bigint): bigint;
export declare function validateWeights(weights: number[]): void;
export declare function calculateInvariant(balances: bigint[], normalizedWeights: number[], decimals: number[]): bigint;
export declare function calcOutGivenIn(params: {
  virtualBalanceIn: bigint; weightInBps: bigint;
  virtualBalanceOut: bigint; weightOutBps: bigint;
  amountIn: bigint; actualBalanceOut: bigint;
}): bigint;
export declare function calcSpotOut(params: {
  virtualBalanceIn: bigint; weightInBps: bigint;
  virtualBalanceOut: bigint; weightOutBps: bigint;
  amountIn: bigint;
}): bigint;
export declare function calcSpotPrice(params: {
  balanceIn: bigint; weightInBps: bigint;
  balanceOut: bigint; weightOutBps: bigint;
  decimalsIn: number; decimalsOut: number;
}): bigint;
export declare function calcBptOutGivenExactTokensIn(actualBalances: bigint[], amountsIn: bigint[], bptTotalSupply: bigint): bigint;
export declare function calcTokensOutGivenBptIn(actualBalances: bigint[], bptAmount: bigint, bptTotalSupply: bigint): bigint[];
export declare function applySlippage(expected: bigint, slippageHbps: number): bigint;
export declare function calculateSwapFee(amount: bigint, swapFeeRate: number): bigint;
export declare function calculateProtocolFee(fee: bigint, protocolFeeRate: number): bigint;
export declare function applySwapFee(amount: bigint, swapFeeRate: number): bigint;
export declare function lpBalances(actual: bigint, virtualBal: bigint, _protocolFeesOwed?: bigint): { lpActual: bigint; lpVirtual: bigint };
export declare function priceImpactHbps(spot: bigint, actual: bigint): number;

export interface SelloffState {
  previousSelloff: bigint; currentSelloff: bigint;
  windowStartTimestamp: bigint; selloffVbSnapshot: bigint;
}
export interface SelloffResult {
  effectiveSelloffBefore: bigint; effectiveSelloff: bigint;
  maxSelloffCap: bigint; vbSnapshot: bigint;
  previousSelloff: bigint; currentSelloff: bigint;
  windowStartTimestamp: bigint;
}
export declare function checkAndAdvanceSelloff(params: {
  state: SelloffState; maxSelloffPct: number; period: number;
  amountIn: bigint; virtualBalance: bigint; now: bigint;
}): SelloffResult | null;
export declare function rescaleSelloffWindow(state: SelloffState, ratio: bigint, isIncrease: boolean, maxSelloffPct: number): SelloffState;

export interface SurgeCurve {
  thresholdPct: number; slopeLowPct: number;
  slopeMidPct: number; slopeHighPct: number; kinkPct: number;
}
export declare function calcSurgeFeePct(params: SurgeCurve & {
  before: bigint; after: bigint; cap: bigint;
}): bigint;
export declare function calcSurgeFeeAmount(params: SurgeCurve & {
  window: SelloffResult | null; amountInAfterFee: bigint; amountOut: bigint;
  virtualBalanceIn: bigint; weightInBps: bigint;
  virtualBalanceOut: bigint; weightOutBps: bigint;
  actualBalanceOut: bigint;
}): bigint;

export interface AllocationResult {
  allocations: bigint[]; wScaled: bigint[]; sumW: bigint;
}
export declare function computeAllocations(params: {
  actualBalances: bigint[]; virtualBalances: bigint[]; weightsBps: number[];
  amountIn: bigint; tokenInIndex: number;
}): AllocationResult;
export declare function computeTwoTokenOptimalAllocations(params: {
  actualBalances: [bigint, bigint]; virtualBalances: [bigint, bigint];
  protocolFeesOwed: [bigint, bigint]; weightsBps: [number, number];
  amountIn: bigint; tokenInIndex: 0 | 1; swapFeeRate: number; protocolFeeRate: number;
}): AllocationResult;
export declare function capDepositAmountsToLpRatio(params: {
  helperBalances: bigint[]; actualBalances: bigint[]; protocolFeesOwed?: bigint[];
}): { depositAmounts: bigint[]; refundAmounts: bigint[]; lpBalancesForAdd: bigint[] };
```

Bounds helpers return the validated input; omitted labels default to `"value"`. Exported bounds include `U64_MAX`, `U128_MAX`, `I128_MIN`, `I128_MAX`, `MIN_NATURAL_EXPONENT`, and `MAX_NATURAL_EXPONENT`. `mulDivDown` performs checked integer multiply/divide; `mulDown/mulUp/divDown/divUp` operate with the fixed-point scale. `weightToFp` converts `10_000`-scaled weights to that scale.

`calcOutGivenIn` expects input **after the base swap fee**; it does not charge the protocol share, advance sell-off state, or deduct the surge fee. `calcTokensOutGivenBptIn` performs proportional output math; the high-level exit quote supplies the minimum-BPT-preserving effective burn. `calculateInvariant` returns raw nine-decimal BPT units. `calcSpotPrice` returns the fixed-point price of one output token in input-token units, adjusted for the supplied decimals; `calcSpotOut` returns a raw output estimate.

`checkAndAdvanceSelloff` takes `now` and the window timestamp as bigint Unix seconds and `period` as numeric seconds. It returns null when the limiter is disabled. `rescaleSelloffWindow.ratio` is fixed-point and `isIncrease` selects deposit versus withdrawal. `calcSurgeFeePct` returns a percent-scale result, while `calcSurgeFeeAmount` returns a raw output-token fee and accounts for nonlinear AMM output across curve segments. The quote composes it with the updated window; applying one endpoint percentage to the entire swap is a different calculation.

`capDepositAmountsToLpRatio` returns the first helper cap; the pool's subsequent join can crop again. Its optional `protocolFeesOwed` field and `lpBalances`' similarly named argument do not subtract fees from v5 actual balances. `computeAllocations` is the deployed STLD allocation primitive. `computeTwoTokenOptimalAllocations` remains an analytical alternative and should not be substituted into a transaction quote.
