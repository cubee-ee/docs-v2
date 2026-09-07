# SDK

`@cubee_ee/sdk` is Coffer's TypeScript library for reading pools, quoting swaps and liquidity operations, and constructing Solana instructions. These pages describe SDK **0.11.1**, source revision `27de819`, against the contracts in `audit-fixes-excluded-SF` at `96a2ee2`. See [version scope](../technical/versions.md) for the source revisions used by this documentation.

The SDK builds unsigned instructions and transactions. Your application supplies the wallet, obtains signatures, submits transactions, and confirms their outcome. `AdminClient.initializeTreasuryIfMissing` is an explicit exception: it can send a transaction through its Anchor provider.

## Installation and configuration

```bash
npm install @cubee_ee/sdk @solana/web3.js bn.js
```

Use the package version matching your deployed contracts. The version documented here identifies the checked source; it does not by itself establish which version your package registry or lockfile resolves.

```ts
import BN from "bn.js";
import { PublicKey } from "@solana/web3.js";
import { CubicPoolClient, getConfig } from "@cubee_ee/sdk";

const config = getConfig("mainnet", {
  rpcEndpoints: [primaryRpcUrl, fallbackRpcUrl],
  rpcTimeoutMs: 2_000,
  rpcCommitment: "confirmed",
  slippageHundredthsBps: 5_000, // 0.5%
});

const pool = new CubicPoolClient({
  config,
  poolAddress: new PublicKey(poolAddress),
});
const synced = await pool.sync();
if (!synced.ok) throw new Error(synced.error.humanMessage);

const quote = pool.quoteSwap(0, 1, new BN("1000000"), 5_000);
if (!quote.ok) throw new Error(quote.error.humanMessage);

const built = pool.buildSwapTx({
  user: walletPublicKey,
  tokenInIndex: 0,
  tokenOutIndex: 1,
  amountIn: quote.data.amountIn,
  minAmountOut: quote.data.minAmountOut,
});
if (!built.ok) throw new Error(built.error.humanMessage);
// built.data.instructions are unsigned. Compile, sign, send, and confirm
// through your application's transaction flow.
```

The example assumes your application supplies the RPC URLs, pool address, and wallet public key. Configuration overrides are **flat**: pass `rpcEndpoints` and `slippageHundredthsBps` directly to `getConfig`, not inside a `defaults` object. The returned `CubeConfig` contains a `defaults` object.

A nonempty `rpcEndpoints` list replaces the endpoint list. A single `rpcEndpoint` is tried before the network's default fallbacks. `getConfig` defaults to confirmed commitment, a two-second RPC timeout, a 1,400,000 compute-unit suggestion, and **5%** quote slippage. Set slippage explicitly for your application.

## Units and state

| Value | Representation and scale |
| --- | --- |
| Token amounts, supplies, virtual balances | `BN`, in raw integer units; use the mint's decimals only for display |
| Math helper amounts | `bigint`; preserve integers when converting to or from `BN` |
| BPT amounts | Raw units of the pool's BPT mint; BPT has nine decimals |
| Weights | `10_000 = 100%`; all weights sum to `10_000` |
| Swap fee and SDK slippage | `1_000_000 = 100%`; `5_000 = 0.5%` |
| Protocol share of the swap fee | `10_000 = 100%`; this is a share of the fee, not of the whole trade |
| Sell-off percentages and fee slopes | `10_000 = 100%`, except `feeKinkPct`, which uses whole percent |
| Range leverage limits | `10_000 = 1×`; zero disables the corresponding bound |
| Quote time override | Unix seconds, not milliseconds |

`sync()` reads the pool, its mints, BPT supply, and Solana Clock. `getCached()` returns the last successful snapshot. Reads are not an atomic snapshot across all accounts, and quotes do not reserve liquidity. Sync again before building a user decision around a quote, and use explicit output floors when signing.

For v5 accounts, `actualBalance` is the LP reserve. Protocol fees are tracked separately. Do not subtract `protocolFeesOwed` from `actualBalance` a second time. The SDK quote path uses the current integer math, fee rounding, sell-off windows, dynamic output fees, proportional deposits, and minimum remaining BPT rules described in [pool mathematics](../technical/math.md).

## Pool and liquidity lifecycle

1. Select an existing config, or have the protocol admin build a config initialization through `PoolFactoryClient.buildInitializeConfigTx`. Config initialization uses the protocol-admin wrapper; the returned config keypair must also sign.
2. Call `PoolFactoryClient.buildDeployPoolTx` to initialize a pool and its BPT mint. Token, weight, and virtual-balance vectors share one order. This step does not seed reserves.
3. The pool admin uses `quoteSeedDeposit` for the first deposit and passes its `minimumBptAmount` into `buildAddLiquidityTx`.
4. Later depositors use `quoteAddLiquidity`. Input amounts are **spend ceilings**. Show `depositAmounts` as the estimated amounts actually deposited; `refundAmounts` remain in the wallet.
5. For an exit, use `quoteRemove`. Its `effectiveBptIn` can be smaller than the request because the pool preserves 1,000 raw BPT. Pass an explicit per-token minimum vector into `buildRemoveLiquidityTx`.

For a one-token entry into an existing pool, see [single-token deposits](single-token-deposit.md). Pool deployment and off-chain metadata registration are separate operations; `CubeBackendClient.createPool` does not deploy a Solana pool.

## Transaction compilation and lookup tables

Builders return `BuiltTx`, containing `instructions`, a `suggestedCuLimit`, and sometimes `extraSigners` public keys. A public key in `extraSigners` is not a private key or a signature. Keep any separately returned keypairs, such as `configKeypair`, in the application's signer flow.

`compileBuiltTx(connection, payer, built, poolInfo)` compiles an unsigned v0 transaction using `poolInfo.lookupTable`. `buildVersionedTx` accepts the instructions and optional lookup-table address directly. A missing or unreadable advertised ALT returns `alt_fetch_failed`; an undefined or zero address means no ALT. Blockhash fetching and message compilation can still throw. These helpers neither create an ALT nor guarantee that every instruction combination fits within the transaction-size limit.

`buildInitializePoolAltTx` builds pool ALT initialization and returns the derived address. Confirm its creation and wait until its addresses are usable in a later slot before compiling dependent transactions. For larger single-token deposits, use separate setup and deposit transactions, with the pool ALT on the deposit.

## Supported tokens and ABI access

Pool token accounts can use classic SPL Token or compatible Token-2022 mints. The SDK reads the actual mint owner and preserves the BPT token program. Admission policy and transaction compatibility are separate checks: a permissive extension bitmap does not add transfer-fee accounting or transfer-hook accounts to this deployed program. Swap/add/remove paths check the token legs involved in their operation. STLD transaction builders check every pool mint, including sidelined tokens, because the helper may return existing balances from any of its ATAs. A STLD quote can therefore succeed while its stricter builder rejects an incompatible sidelined mint. Do not promise arbitrary Token-2022 support merely because a mint passed a configurable ban list.

The exported `IDLS`, `ContractInstructionMap`, `ContractAccountMap`, and `ContractEventMap` cover the three programs. `buildContractInstruction` exposes every instruction with typed arguments and explicit accounts; `decodeContractAccount`, `decodeContractEvent`, and `parseContractEvents` decode the complete current ABI. These APIs complement the convenience clients. See the [SDK reference](reference.md) and [contract instruction reference](../technical/instruction-reference.md).

## Results, errors, and backend integration

High-level quotes and most client operations return `SdkResult<T>`: check `ok` before reading `data`; failures contain `error.code` and `error.humanMessage`. Raw builders, math helpers, decoders, and Anchor instruction builders can throw. Do not assume that every exported function returns `SdkResult` or that successful construction proves a transaction will execute.

The backend client has a separate constructor:

```ts
import { CubeBackendClient } from "@cubee_ee/sdk";

const backend = new CubeBackendClient({ apiEndpoint: backendUrl });
const pools = await backend.listPoolsRaw(20, 0);
```

Backend data powers discovery, analytics, portfolios, referrals, and XP. It is not the authority for on-chain balances or contract authorization. SDK methods and a deployed backend may have different release schedules: check the [REST availability table](../integration/api-reference.md) before relying on a route or response field.
