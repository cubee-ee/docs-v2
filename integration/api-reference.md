# REST API reference

Coffer's backend provides indexed pool metadata, routing estimates, analytics, wallet authentication, portfolio views, referrals, and XP. This page compares SDK 0.11.1 (`09cc776`) with the **local backend source** on branch `v5.1` at `a886497`. It is not a live-server inventory. A deployed backend may run another revision; verify its version and response shape before relying on an SDK method.

Use your deployment's base URL as `apiEndpoint` when constructing `CubeBackendClient`. Endpoint paths below are relative to that URL. See [version scope](../technical/versions.md) and the [complete SDK method reference](../sdk/reference.md#cubebackendclient).

## Authentication and response conventions

Public pool, statistics, routing, and leaderboard reads do not require a global `X-Cube-Api-Key` in this backend revision. Protected portfolio/referral/pool-admin endpoints use `Authorization: Bearer <accessToken>`. Internal admin and Layer3 integrations have separate key mechanisms; those keys are not a replacement for a wallet's JWT.

Responses are **not uniformly wrapped**. Pool discovery generally returns `{ success: true, data: ... }`, paginated pools add `hasMore` and `totalCount`, and statistics/leaderboard list endpoints can return flat objects. Errors commonly use Nest's `{ statusCode, message, error }` response. Check status and actual JSON instead of assuming a universal success/error envelope.

The SDK's `listPoolsRaw` and `getPoolRaw` normalize the pool envelopes. `listPools` and `getPool` have more optimistic declared TypeScript types and do not perform equivalent unwrapping. Generic HTTP methods preserve JSON structure and do not validate an asserted `<T>`.

### Wallet sign-in

| Method and path | Input | Result |
| --- | --- | --- |
| `GET /api/auth/nonce` | Query `wallet`: base58 wallet address | `{ nonce, message }` with the exact Sign In With Solana message to sign |
| `POST /api/auth/verify` | JSON `{ message, signature }`; signature is **base64-encoded ed25519 bytes** | `{ accessToken, refreshToken, wallet, expiresIn }` |
| `POST /api/auth/refresh` | JSON `{ refreshToken }` | A new access/refresh pair and wallet/expiry fields |

Sign the exact returned message. Verification checks its domain, URI, chain ID, address, issued/expiration times, nonce, and signature. A nonce is consumed during verification, so obtain a fresh challenge after a failed or expired attempt rather than replaying it.

```ts
import { CubeBackendClient } from "@cubee_ee/sdk";

const backend = new CubeBackendClient({ apiEndpoint: backendUrl });
const challenge = await backend.getNonce(walletAddress);
if (!challenge.ok) throw new Error(challenge.error.humanMessage);
// Ask the connected wallet to sign challenge.data.message.
// Convert the returned signature bytes to base64, preserving the message.
const auth = await backend.verifySignature(
  challenge.data.message,
  signatureBase64,
);
if (!auth.ok) throw new Error(auth.error.humanMessage);
backend.setTokens(auth.data.accessToken, auth.data.refreshToken);
```

The application supplies `backendUrl`, the wallet address, and the signature; the SDK does not open a signing dialog itself. It keeps tokens in memory, retries a non-auth request once after a successful 401 refresh, and exposes callbacks for persistence or reauthentication. Transaction-based sign-in is declared by the SDK but not implemented in the checked backend source; see the availability table below.

## Pool discovery and metadata

These GET routes are public:

| Path | Query | Response content / SDK method |
| --- | --- | --- |
| `/api/pools` | `limit` default 50, range 1–100; `offset` default 0, range 0–10,000 | `{ success, data: pools[], hasMore, totalCount }`; `listPoolsRaw(limit, offset)` |
| `/api/pools/my` | `wallet`, same `limit/offset` | Pools indexed for that admin wallet; generic `get` |
| `/api/pools/tokens` | None | Token catalog envelope; `getAllTokens<T>()` |
| `/api/pools/top-tokens` | `limit`; server default 30, bounded to 1–100 | Token catalog envelope; SDK `getTopTokens<T>(limit = 20)` sends its own default |
| `/api/pools/by-pair` | `tokenA`, `tokenB` mint addresses | Pools containing the pair; `getPoolsByTokenPair<T>` |
| `/api/pools/portfolio` | `wallet` | Public pool/BPT holdings lookup; `getPortfolio<T>(wallet)` |
| `/api/pools/stats` | None | Platform actual/virtual TVL, 24h volume, pool count, update time; `getPlatformStats()` unwraps data |
| `/api/pools/:address` | None | Single-pool envelope; `getPoolRaw(address)` |
| `/api/pools/:address/transactions` | `limit`, `offset`, optional `type` and `user` | Indexed transaction rows; `getTransactions<T>` |
| `/api/pools/:address/tx-stats` | None | Pool transaction aggregates; `getPoolTxStats<T>` |

Addresses and raw token amounts in REST JSON are strings. Display amounts, prices, and analytics can be ordinary numbers. `PoolSummary` is an SDK-facing model, not a guarantee that every backend field has that exact structure. Metadata and historical metrics are indexed and can lag the chain.

| Method and path | Input | Authorization and behavior |
| --- | --- | --- |
| `POST /api/pools/register` | `{ poolAddress, poolName, adminWallet }`; pool name is 1–12 alphanumeric characters | Registers an already initialized pool after backend on-chain validation. Use generic `post`; there is no dedicated `registerPool` SDK method |
| `POST /api/pools` | Legacy full pool metadata DTO, including token metadata | `createPool<T>(body)` targets this route. It writes backend records and does not deploy a contract |
| `GET /api/pools/admin/my-pools` | None | JWT; `getAdminPools()` |
| `GET /api/pools/admin/is-admin/:address` | None | JWT; `isPoolAdmin(address)` |
| `PUT /api/pools/admin/:address/name` | `{ name }`, 2–64 characters without control characters | JWT plus the pool-admin check; `renamePool(address, name)` |

Registering metadata and changing an on-chain pool parameter are different operations. Backend authorization for a name does not substitute for a contract authority's signature. The checked registration service also performs program/bytecode validation using its configured reference; keep that backend reference in step with a contract upgrade.

## Routing estimates

`GET /api/pools/swap-route` accepts:

| Query | Meaning |
| --- | --- |
| `tokenIn` | Base58 input mint |
| `tokenOut` | Base58 output mint |
| `amountIn` | Raw input as a decimal digit string, up to 20 digits |
| `decimalsIn` | Input display decimals, 0–18; default 9 |

The checked controller returns `{ success: true, data: { routes, totalAmountIn, totalExpectedOut, effectivePrice, priceImpact, spotPrice, feePercent, estimatedXp, xpBoostApplied } }`. Route amounts are decimal strings. Each route includes pool address/name, allocated `amountIn`, `expectedOut`, allocation `percentage`, `swapFee`, token program addresses, token indices, and nullable vault addresses. Price impact and fee percent are displayed in percent, not the SDK's hundredths-bps units.

SDK `getSwapRoute` also accepts `slippageBps` and `pool`, and its response type declares per-leg `minAmountOut`, total `minReceived`, and `slippageBps`. **Those options/fields are not implemented by this checked controller/DTO.** Do not pass an undefined backend floor into an instruction or assume single-pool filtering works on every backend release.

This backend routing implementation does not reproduce all v5 state-dependent rules. Read [swap routing](swap-routing.md) before using allocations to build transactions. A displayed route is an estimate, not a signed transaction or a guarantee of executable output.

## Time-series statistics

`GET /api/stats/:metric` returns the flat object `{ points: [{ t, v }] }`, where `t` is a Unix timestamp in **milliseconds**. It reads ten-minute stored buckets in ascending order.

| Parameter | Allowed values |
| --- | --- |
| `metric` | `tvl`, `volume`, `swap_count`, `avg_swap`, `median_swap`, `fees_lp`, `fees_protocol`, `users_total`, `dau`, `mau`, `deposits`, `removals` |
| `window` | `1d`, `7d` (default), `30d`, `all` |
| `pool` | Pool address; omitted or `all` selects protocol aggregates |

In this implementation, `all` means a **365-day lookback**. Monetary metrics use USD values. The SDK's `unit` argument is not consumed by this controller and does not convert the series to raw token units. Unavailable or stale indexed data should be reflected in the UI rather than presented as an on-chain guarantee.

## Authenticated portfolio and referral routes

These routes use the wallet from the JWT, not a caller-supplied wallet query:

| Method and path | Parameters | SDK method |
| --- | --- | --- |
| `GET /api/portfolio/summary` | None | `getPortfolioSummary()` |
| `GET /api/portfolio/exposure` | None | `getPortfolioExposure()` |
| `GET /api/portfolio/history` | `range`: `7d`, `30d`, `90d`, `all` | `getPortfolioHistory(range)` |
| `GET /api/portfolio/pools` | None | `getPortfolioPools()` |
| `GET /api/portfolio/pools/:address/history` | Same `range` | `getPortfolioPoolHistory(address, range)` |
| `POST /api/referral/bind` | `{ code, utm? }`; optional source/medium/campaign/content/term | `bindReferral(code, utm)` |
| `GET /api/referral/my` | None | `getReferralStatus()` |
| `GET /api/referral/my/referrals` | `page`, `limit` | `getMyReferrals(page, limit)` |

Portfolio responses contain valuation and history estimates derived from holdings and indexed prices. They are not a guarantee of realizable proceeds, exact impermanent loss, or future yield. Referral binding and bonuses are implemented in this backend revision; see [Cube XP](../rewards/cube-xp.md).

## XP leaderboard

| Public GET path | Query | Response |
| --- | --- | --- |
| `/api/leaderboard` | `page`, `limit` | Flat `{ total, page, limit, data: [{ address, points, place }] }` |
| `/api/leaderboard/stats` | None | `{ totalUsers, totalXp }` |
| `/api/leaderboard/epoch` | None | Epoch dates, countdown, multipliers, base/current rates, and epoch history |
| `/api/leaderboard/user/:address` | None | `{ success: true, data: userStats }`; 404 if unknown |
| `/api/leaderboard/user/:address/history` | `page`, `limit` | Flat paginated accrual rows |

XP is accrued in three-hour backend jobs. The field `swapXpPerUsdLpFee` retains a legacy name: the checked calculation uses **total pool swap fee USD**, including the protocol share. History's `swapVolumeUsd` and the last accrual's swap-USD field similarly store the fee-USD base in this implementation. The LP rate is per three-hour accrual, not per day. See [Cube XP](../rewards/cube-xp.md) for exact rates and timing.

## SDK routes not present in the checked backend revision

The following SDK 0.11.1 methods construct requests, but no corresponding controller route was found in backend `v5.1` at `a886497`. This is a **source-version gap**, not a statement about what any live deployment currently serves. Coordinate releases or check deployed support before enabling these features.

| SDK method(s) | Declared route |
| --- | --- |
| `getTxChallenge`, `verifyTransaction` | `GET /api/auth/tx-challenge`, `POST /api/auth/verify-tx` |
| `getTokenPrices` | `GET /api/prices` |
| `getTokenPairChart` | `GET /api/tokens/pair-chart` |
| `updatePoolSettings` | `PUT /api/pools/admin/:address/settings` |
| `getPortfolioChart` | `GET /api/portfolio/chart` |
| `getPortfolioActivity` | `GET /api/portfolio/activity` |
| `getPortfolioHoldings` | `GET /api/portfolio/holdings` |
| `getPortfolioPositions` | `GET /api/portfolio/positions` |
| `joinCampaign` | `POST /api/campaign/join` |
| `getCampaignStatus`, `getCampaignInfo` | `GET /api/campaign/me`, `GET /api/campaign/info` |
| `getCampaignRank`, `getCampaignTop` | `GET /api/campaign/me/rank`, `GET /api/campaign/top` |

Availability of `/api/pools/swap-route` also does not imply support for the SDK's newer `pool`/`slippageBps` options or floor fields. Inspect real responses and use the synced SDK pool quote for v5 transaction constraints.

## Operational behavior

The backend includes IP-based throttling and browser CORS rules; deployment configuration and endpoint-specific policies affect access. A browser CORS allowance is not authentication. Handle 400 validation errors, 401 authentication failures, 403 authorization failures, 404 missing resources, 429 throttling, and temporary server/RPC failures explicitly.

`GET /health` returns health status and a timestamp; `GET /api/version` returns backend version information and a timestamp. Health alone does not prove that every indexed dataset is current or that the backend's reference bytecode matches an upgraded program.

Sources: [pool routes](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/pool/pool.controller.ts),
[sign-in verification](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/auth/auth.service.ts),
[statistics routes](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/stats/stats.controller.ts),
[SDK backend client](https://github.com/coffer-so/sdk/blob/09cc7766a1e865b5c3f9b97a0526a982671683f5/src/clients/CubeBackendClient.ts).
