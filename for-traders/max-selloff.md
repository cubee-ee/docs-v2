# Max-Selloff Window

Every Cube pool can rate-limit how much of each token can be **sold INTO** the pool over a configurable time window. When a swap would push the rolling sum above the cap, the swap reverts with `MaxSelloffExceeded` and the user's funds stay put.

This page explains what the parameter is, how it's enforced on-chain, and how it should look from a trader's perspective.

---

## TL;DR

- Each token has two on-chain fields: **`max_selloff`** (a cap, in raw token units) and **`max_selloff_period_length`** (a window in seconds).
- The pool tracks a **sliding window** of how much of that token has been sold into the pool over the last `period` seconds.
- A new swap is allowed only if the effective rolling sum (after adding `amount_in`) stays ≤ `max_selloff`.
- `max_selloff = 0` → check is **disabled** for that token.
- Set by `pool_admin` only (see [Pool Controls](../safety/pool-controls.md)).

---

## Why this exists

A weighted-product AMM with virtual liquidity is **very generous to one-sided flow**: if everyone wants to dump token X into the pool, the pool happily absorbs it — pricing X lower with every fill, eventually leaving the LP holding a bag of the dumping token at a curve-implied price well below market.

`max_selloff` is a **circuit breaker** against this. The pool admin caps how fast each token can be dumped in, giving the range-manager (or external arbitrage) time to either rebalance or pause the pool before too much damage accrues.

It is **not** a fee or a discouragement — once you hit the cap, you just can't trade that direction until the window slides forward enough.

---

## How the math works

The pool stores three numbers per token, updated atomically on every swap:

| Field | Meaning |
|---|---|
| `previous_selloff` | Total `amount_in` accrued in the PREVIOUS window |
| `current_selloff` | Total `amount_in` accrued in the CURRENT window so far |
| `window_start_timestamp` | Unix-second the current window began |

When a new swap arrives at time `now`:

```
elapsed = now − window_start_timestamp
```

### Bucket rotation

Before checking the cap, the pool decides whether the window has rolled over:

| Condition | Action |
|---|---|
| `elapsed ≥ 2 × period` | Both buckets stale → wipe everything, start fresh at `now`. |
| `elapsed ≥ period` | One boundary crossed → `previous := current`, `current := 0`, slide window by exactly one period. |
| `elapsed < period` | No rotation. |

This is a Cloudflare-style sliding-window approximation (the same pattern Solend's rate limiter uses).

### The cap check

The "effective" amount on a given swap blends both buckets with a linear weight:

```
effective = previous × (period − elapsed_in_window) / period
          + current
          + amount_in

require: effective ≤ max_selloff
```

The linear weight on `previous` means old activity fades out smoothly across the new window — so you can't game the boundary by waiting until `00:00` to dump exactly `max_selloff`, then dumping another `max_selloff` at `00:01`. The early-window cap is the **combination** of leftover-previous + fresh-current.

If `effective > max_selloff` the instruction reverts with `MaxSelloffExceeded` and **no state is mutated** — `dynamics` stays exactly as it was. Retry with a smaller `amount_in` or wait.

### After accept

```
current += amount_in
```

That's the only mutation. `previous` and `window_start_timestamp` only change during rotation.

---

## Worked example

Pool: 9-token, USDC/SOL/JTO/.../BONK. Admin sets for BONK:

```
max_selloff               = 1_000_000      (raw, BONK has 5 decimals → 10 BONK)
max_selloff_period_length = 3600           (1-hour window)
```

| t | swap | effective calc | accept? | state after |
|---|---|---|---|---|
| 0 | — | — | (init) | `prev=0 curr=0 ws=0` |
| 10s | sell 5 BONK in (`5_000_000` raw) | `0·… + 0 + 5_000_000 = 5_000_000` | ❌ over 1M | unchanged |
| 10s | sell 0.5 BONK in (`500_000`) | `0·… + 0 + 500_000 = 500_000` | ✅ | `curr=500_000` |
| 1800s | sell 0.4 BONK in (`400_000`) | `0 + 500_000 + 400_000 = 900_000` | ✅ | `curr=900_000` |
| 3700s | sell 0.5 BONK in | elapsed=3700>3600 → rotate: `prev=900_000 curr=0 ws=3600`<br>elapsed_in_window=100<br>`weighted_prev = 900_000 × (3600−100)/3600 = 875_000`<br>`effective = 875_000 + 0 + 500_000 = 1_375_000` | ❌ over 1M | unchanged |
| 10000s | sell 0.5 BONK in | elapsed=10000 > 2×3600 → hard reset, `ws=10000`<br>`effective = 500_000` | ✅ | `prev=0 curr=500_000 ws=10000` |

---

## What a trader sees

If your swap would push the pool over the cap, the wallet shows a simulation error:

```
Transaction simulation failed: Error processing Instruction N: custom program error: 0x???
… Program log: AnchorError caused by account: pool. Error Code: MaxSelloffExceeded.
```

(The SDK and frontend translate this into "Max-selloff window exceeded for &lt;TICKER&gt;. Try a smaller amount or wait for the window to roll forward.")

### How to react

1. **Retry with smaller `amount_in`.** Compute headroom = `max_selloff − effective(before your amount_in)`. The pool exposes `previous_selloff`, `current_selloff`, and `window_start_timestamp` on-chain so you (or your bot) can pre-compute this.
2. **Wait.** As time passes, `weighted_prev` shrinks linearly. After exactly one full period the previous window has fully aged out and you regain full headroom (minus whatever has accumulated in the new `current`).
3. **Split across a different pool.** Cube's [Swap Routing](../integration/swap-routing.md) backend will automatically route around a capped pool when picking a multi-hop path.

### How to estimate the cap before submitting

```typescript
const pool = await client.sync();
const slot = pool.data.tokens[tokenInIndex];
const cap = slot.maxSelloff;
const period = slot.maxSelloffPeriodLength;
if (cap === 0n) return; // disabled — no limit

const now = Math.floor(Date.now() / 1000);
const elapsed = Math.max(0, now − slot.windowStartTimestamp);
let prev = slot.previousSelloff;
let curr = slot.currentSelloff;
let elapsedInWindow = elapsed;
if (elapsed >= 2 * period) { prev = 0n; curr = 0n; elapsedInWindow = 0; }
else if (elapsed >= period) {
  prev = curr; curr = 0n;
  elapsedInWindow = elapsed − period;
}
const weightedPrev = (prev * BigInt(period − elapsedInWindow)) / BigInt(period);
const headroom = cap − (weightedPrev + curr);
```

---

## Configuration (admin side)

Pool admin sets per-token caps via `set_max_selloff` on the cubic-pool program:

```rust
set_max_selloff(
    ctx: Context<SetMaxSelloff>,
    max_selloffs: Vec<u64>,    // length = token_count, in RAW units
    period_lengths: Vec<u32>,  // length = token_count, seconds
)
```

Both vectors must be exactly `token_count` long — sparse updates are not supported. To leave a token unchanged, pass its current values. `max_selloff = 0` disables the check for that slot (period is ignored).

Frontend exposes this as a form on **Admin panel → Set max-selloff window**. Inputs are in **human units** (e.g. "10" BONK), the page converts to raw `u64` via `10^decimals` before sending.

See [Pool Controls](../safety/pool-controls.md#set-max-selloff-window) for the full admin UI walkthrough.

---

## Reference

- Implementation + tests: `programs/cubic-pool/src/math/max_selloff.rs`
- Set instruction: `programs/cubic-pool/src/instructions/set_max_selloff.rs`
- Event: `MaxSelloffWindowAdvanced` — emitted on every swap that passed the check, carrying the new effective state. Use for monitoring dashboards.
- Error: `MaxSelloffExceeded` (Anchor code defined in `programs/cubic-pool/src/errors.rs`).
