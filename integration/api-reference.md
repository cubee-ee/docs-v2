# API Reference

The Cube backend exposes a REST API for pool discovery, swap routing,
statistics, leaderboard, and data retrieval. Pool endpoints sit under
`/api/pools`; time-series statistics under `/api/stats`; leaderboard
under `/api/leaderboard`.

Base URL: `https://api.cubee.ee`

---

> 🔐 **API key required.** The Cube backend is gated. Before integrating, request a key:
>
> - Telegram chat: **[@cubee\_chat](https://t.me/cubee_chat)**
> - Direct: **[@sepezho](https://t.me/sepezho)**
>
> Pass the key as a header on every request:
>
> ```
> X-Cube-Api-Key: <your-key>
> ```
>
> The on-chain program is permissionless — anyone can call its instructions directly via RPC. The API gating only applies to the convenience backend that wraps RPC reads with prices, TVL, indexed transactions, and pool metadata.

---

> **Tip** — You probably don't need this reference directly. The
> [@cube/sdk](../sdk/index.md) ships a typed `CubeBackendClient` that
> wraps every endpoint listed here, returns `SdkResult<T>` instead of
> throwing, and handles retry + human-readable error mapping. Frontend
> and backend integrations should consume the SDK rather than calling
> these endpoints directly. The SDK reads the API key from the
> `CubeConfig` you pass to `getConfig({ apiKey })` and sends it
> automatically.

---

## Pool Discovery

### List Recent Pools

```
GET /api/pools?limit=50&offset=0
```

Returns the most recent pools, ordered by pinned status then creation date.

**Query Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 50 | Results per page (1–100) |
| `offset` | integer | 0 | Pagination offset (0–10,000) |

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "poolAddress": "ABC...xyz",
      "poolName": "SOL-USDC",
      "adminWallet": "DEF...uvw",
      "swapFee": 0.3,
      "protocolFeeRate": 3.0,
      "poolEnabled": true,
      "swapsEnabled": true,
      "tvlUsd": 125000.50,
      "virtualTvlUsd": 312500.00,
      "volume24h": 45000.00,
      "apy": 12.5,
      "bptMint": "GHI...rst",
      "bptTotalSupply": "1000000000",
      "poolAdmin": "ABC...xyz",
      "pendingPoolAdmin": null,
      "rangeManager": "RNG...mgr",
      "rangeManagerEnabled": true,
      "rangeManagerMaxVbChangeBps": 500,
      "rangeManagerMaxWeightChangeBps": 500,
      "rangeManagerMinUpdateIntervalSecs": 60,
      "rangeManagerLastUpdated": "1748044800",
      "lookupTable": "ALT...lut",
      "tokens": [
        {
          "mintAddress": "So11111111111111111111111111111111111111112",
          "ticker": "SOL",
          "decimals": 9,
          "weight": 50,
          "leverage": 2,
          "actualBalance": "1000000000000",
          "virtualBalance": "2000000000000",
          "protocolFeesOwed": "0",
          "maxSelloff": "0",
          "maxSelloffPeriodLength": 0,
          "tokenProgram": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
          "vaultAddress": "JKL...opq",
          "orderIndex": 0,
          "imageUrl": "/img/solana.png"
        },
        {
          "mintAddress": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
          "ticker": "USDC",
          "decimals": 6,
          "weight": 50,
          "leverage": 2,
          "actualBalance": "1000000000",
          "virtualBalance": "2000000000",
          "vaultAddress": "MNO...lmn",
          "orderIndex": 1,
          "imageUrl": "https://..."
        }
      ]
    }
  ],
  "hasMore": false,
  "totalCount": 1
}
```

### Get Pool by Address

```
GET /api/pools/:address
```

Returns a single pool with all token data.

**Path Parameters:**

| Name | Type | Description |
|---|---|---|
| `address` | string | Pool's on-chain address (base58, 32–44 chars) |

### Get My Pools

```
GET /api/pools/my?wallet=<address>&limit=50&offset=0
```

Returns pools created by a specific wallet.

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `wallet` | string | Yes | Admin wallet address (base58) |
| `limit` | integer | No | Results per page (1–100) |
| `offset` | integer | No | Pagination offset |

### Get Pools by Token Pair

```
GET /api/pools/by-pair?tokenA=<mint>&tokenB=<mint>
```

Returns all pools containing both specified tokens, ordered by TVL descending.

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tokenA` | string | Yes | First token mint address (base58) |
| `tokenB` | string | Yes | Second token mint address (base58) |

### Get Portfolio Pools

```
GET /api/pools/portfolio?wallet=<address>
```

Returns all pools with BPT information for portfolio tracking. If `wallet` is provided, each pool includes an `isCreator` flag.

---

## Token Discovery

### List All Tokens

```
GET /api/pools/tokens
```

Returns all unique tokens across active pools with their metadata.

**Response item:**

```json
{
  "mintAddress": "So11111111111111111111111111111111111111112",
  "ticker": "SOL",
  "decimals": 9,
  "tokenProgram": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
  "priority": 100,
  "imageUrl": "/img/solana.png",
  "pools": [
    { "poolAddress": "...", "poolName": "SOL-USDC", "tvlUsd": 125000 }
  ]
}
```

### Get Top Tokens

```
GET /api/pools/top-tokens?limit=30
```

Returns top tokens by priority and pool count. Useful for UI token selectors.

---

## Swap Routing

### Calculate Swap Route

```
GET /api/pools/swap-route?tokenIn=<mint>&tokenOut=<mint>&amountIn=<amount>&decimalsIn=9
```

Calculates the optimal swap route across all available pools. This is the primary endpoint for integrators building swap interfaces.

**Query Parameters:**

| Name | Type | Required | Default | Description |
|---|---|---|---|---|
| `tokenIn` | string | Yes | — | Input token mint address (base58) |
| `tokenOut` | string | Yes | — | Output token mint address (base58) |
| `amountIn` | string | Yes | — | Input amount in native units (positive integer, max 20 digits) |
| `decimalsIn` | integer | No | 9 | Input token decimals (0–18) |

**Response:**

```json
{
  "success": true,
  "data": {
    "routes": [
      {
        "poolAddress": "ABC...xyz",
        "poolName": "SOL-USDC",
        "amountIn": "500000000",
        "expectedOut": "74925000",
        "percentage": 50.0,
        "swapFee": 0.3,
        "tokenInIndex": 0,
        "tokenOutIndex": 1,
        "tokenProgramIn": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
        "tokenProgramOut": "TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBULCykXv4t2",
        "vaultIn": "JKL...opq",
        "vaultOut": "MNO...lmn"
      },
      {
        "poolAddress": "DEF...uvw",
        "poolName": "SOL-USDC-2",
        "amountIn": "500000000",
        "expectedOut": "74800000",
        "percentage": 50.0,
        "swapFee": 0.3,
        "tokenInIndex": 0,
        "tokenOutIndex": 1,
        "tokenProgramIn": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
        "tokenProgramOut": "TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBULCykXv4t2",
        "vaultIn": "PQR...ijk",
        "vaultOut": "STU...ghi"
      }
    ],
    "totalAmountIn": "1000000000",
    "totalExpectedOut": "149725000",
    "effectivePrice": 0.149725,
    "priceImpact": 0.18,
    "spotPrice": 0.15
  }
}
```

**Response Fields:**

| Field | Type | Description |
|---|---|---|
| `routes` | array | Per-pool route splits |
| `routes[].poolAddress` | string | Pool on-chain address |
| `routes[].poolName` | string | Pool display name |
| `routes[].amountIn` | string | Input amount allocated to this pool (native units) |
| `routes[].expectedOut` | string | Expected output from this pool (native units) |
| `routes[].percentage` | number | Percentage of total input routed here |
| `routes[].swapFee` | number | Pool's swap fee (display %, e.g., 0.3) |
| `routes[].tokenInIndex` | number | Input token index in the pool |
| `routes[].tokenOutIndex` | number | Output token index in the pool |
| `routes[].tokenProgramIn` | string | Token program for input mint |
| `routes[].tokenProgramOut` | string | Token program for output mint |
| `routes[].vaultIn` | string | Pool vault address for input token |
| `routes[].vaultOut` | string | Pool vault address for output token |
| `totalAmountIn` | string | Total input (should match request) |
| `totalExpectedOut` | string | Total expected output across all pools |
| `effectivePrice` | number | Output per input unit (includes fees + impact) |
| `priceImpact` | number | Price impact as percentage (positive = loss) |
| `spotPrice` | number | Weighted average spot price (no-trade reference) |

---

## Platform Stats

### Get Platform Stats

```
GET /api/pools/stats
```

Returns aggregate platform metrics.

**Response:**

```json
{
  "success": true,
  "data": {
    "totalTvlUsd": 500000.00,
    "totalVirtualTvl": 2500000.00,
    "poolCount": 15,
    "updatedAt": "2025-03-15T12:00:00Z"
  }
}
```

---

## Transactions

### Get Pool Transactions

```
GET /api/pools/:address/transactions?limit=50&offset=0&user=<wallet>
```

Returns paginated transactions for a specific pool.

**Query Parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 50 | Results per page (1–100) |
| `offset` | integer | 0 | Pagination offset |
| `user` | string | — | Filter by user wallet address (optional) |

**Response item:**

```json
{
  "id": 1,
  "signature": "5KtPn...",
  "txType": "swap",
  "userAddress": "ABC...xyz",
  "tokenInMint": "So11...",
  "tokenOutMint": "EPjF...",
  "amountIn": "1000000000",
  "amountOut": "149725000",
  "feeAmount": "3000000",
  "protocolFeeAmount": "600000",
  "blockTime": "2025-03-15T12:30:00Z",
  "slot": "123456789"
}
```

Transaction types: `swap`, `add_liquidity`, `remove_liquidity`

### Get Transaction Stats

```
GET /api/pools/:address/tx-stats
```

Returns transaction count summary.

**Response:**

```json
{
  "success": true,
  "data": {
    "totalCount": 1500,
    "count24h": 87
  }
}
```

---

## Pool Registration

### Register Pool (from chain)

```
POST /api/pools/register
```

Registers an on-chain pool with the backend. All pool data is fetched from the blockchain — only minimal identification is needed from the frontend.

**Request Body:**

```json
{
  "poolAddress": "ABC...xyz",
  "poolName": "SOL-USDC",
  "adminWallet": "DEF...uvw"
}
```

| Field | Type | Description |
|---|---|---|
| `poolAddress` | string | On-chain pool address (base58) |
| `poolName` | string | Display name (1–12 chars, alphanumeric) |
| `adminWallet` | string | Wallet claiming ownership (base58); must match an on-chain authority |

**Response:** `201 Created` with the full pool object.

The backend does not trust the claimed admin. It fetches the pool config from
chain and accepts `adminWallet` only if it matches the pool's
`pool_admin`, `protocol_admin`, or the protocol-fees
treasury admin when that treasury controls the config. It also verifies the
on-chain program bytecode, fetches all pool state, derives vault addresses, and
fetches token metadata from Metaplex using public-HTTPS SSRF and response-size
guards.

---

## Health & Version

### Health Check

```
GET /health
```

Returns `{ "status": "ok", "timestamp": "..." }`

### Version

```
GET /api/version
```

Returns `{ "version": "...", "timestamp": "..." }`

---

## Rate Limits

Cube has separate rate limits for the official frontend (`cubee.ee`) and external integrators.

### External Integrators

| Scope | Limit | Description |
|---|---|---|
| Global | 5 req/sec per IP | All endpoints |
| Burst | 100 req/min per IP | 1-minute sliding window |
| GET endpoints | 5 req/sec per IP | Pool queries |

### Frontend (cubee.ee)

| Scope | Limit | Description |
|---|---|---|
| Global | 10 req/sec per IP | All endpoints |
| Burst | 200 req/min per IP | 1-minute sliding window |
| GET endpoints | 10 req/sec per IP | Pool queries |
| POST pool creation | 1 req/2sec per IP+wallet | Prevents spam |

### CORS Policy

- **GET endpoints** are open to all origins (public API).
- **POST endpoints** (pool creation/registration) are restricted to the Cube frontend (`cubee.ee`) via CORS. External integrators cannot create pools via the API.

Rate-limited responses return `429 Too Many Requests` with a `retryAfter` header.

---

## Error Responses

All errors follow this format:

```json
{
  "success": false,
  "error": "Human-readable error message"
}
```

Common HTTP status codes:

| Code | Meaning |
|---|---|
| `400` | Invalid request parameters |
| `404` | Pool or resource not found |
| `409` | Resource already exists (duplicate pool) |
| `429` | Rate limited |
| `500` | Internal server error |

---

## Statistics

Statistics endpoints serve pre-aggregated time-series rolled up at
10-minute granularity.

### Get a metric series

```
GET /api/stats/:metric?pool=<addr|all>&window=<1d|7d|30d|all>&unit=<usd|token>
```

Returns a time-bucketed series.

**Path parameters:**

| Name | Description |
|---|---|
| `metric` | one of: `tvl`, `volume`, `swap_count`, `avg_swap`, `median_swap`, `fees_lp`, `fees_protocol`, `users_total`, `dau`, `mau`, `deposits`, `removals` |

**Query parameters:**

| Name | Type | Default | Description |
|---|---|---|---|
| `pool` | string | `all` | Pool address. `all` for the protocol-wide rollup. |
| `window` | enum | `7d` | One of `1d` / `7d` / `30d` / `all`. |
| `unit` | enum | `usd` | `usd` or `token`. Ignored for unit-less metrics like `swap_count`. |

**Response:**

```json
{
  "points": [
    { "t": 1714003200000, "v": 1234567.89 },
    { "t": 1714003800000, "v": 1240345.12 }
  ]
}
```

`t` is a unix epoch in milliseconds; `v` is the metric value.

SDK:

```ts
const r = await sdkBackend().getStats("tvl", "30d", poolAddr, "usd");
if (r.ok) console.log(r.data.points);
```

---

## Leaderboard

XP leaderboard for the Cube reward program. See [Cube XP](../rewards/cube-xp.md) for the full reward model — these endpoints are the read-side of it.

### Get leaderboard (paginated)

```
GET /api/leaderboard?page=1&limit=20
```

Returns a paginated list of wallets ranked by cumulative XP points (descending).

**Response:**

```json
{
  "total": 847,
  "page": 1,
  "limit": 20,
  "data": [
    {
      "address": "6NAL3YafKj9NPv3bkvdTxph33VGM9ayoJRNaTeBhTAaz",
      "points": 152340.5,
      "place": 1
    }
  ]
}
```

### Get epoch info

```
GET /api/leaderboard/epoch
```

Returns everything needed to render the epoch widget: current epoch number, ms-until-next-epoch (for client-side countdown), base + current XP rates, halving multiplier, and the full history of past epochs.

### Get user XP details

```
GET /api/leaderboard/user/:address
```

Returns the user's current rank, accumulated points, and per-source breakdown (swap vs LP, per pool).

### Get user XP history

```
GET /api/leaderboard/user/:address/history?from=<ts>&to=<ts>
```

Returns per-day XP accrual entries over a window. Used by the user profile page to render the points-over-time chart.

---

## DefiLlama Adapter

Public endpoints used by DefiLlama and similar dashboards. **Not gated** — these don't require an API key (DefiLlama scrapes them anonymously).

```
GET  /api/defillama/dimensions   # Volume + fees + revenue by chain/day
GET  /api/defillama/tvl          # Current TVL snapshot
GET  /api/defillama/yields       # Per-pool APY series
GET  /api/defillama/info         # Protocol metadata
GET  /api/defillama/diag         # Diagnostic — adapter health
```

These follow DefiLlama's adapter spec and are documented at `defillama.com/protocol/cube`.

---

## Webhooks (internal)

```
POST /internal/webhooks/helius
```

Reserved for the indexer's Helius webhook ingest path. **Not a public endpoint** — restricted by a separate webhook secret, not the API key. Listed here only to disambiguate; integrators don't call it.
