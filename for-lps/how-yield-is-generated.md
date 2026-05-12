# How Yield is Generated

Cube LPs earn yield from **swap fees** — there is no staking, farming,
or external reward mechanism. Yield is generated passively as traders
use the pool.

---

## Fee mechanics

Every swap charges a fee on the input token. The fee is split:

| Recipient | Default share | Notes |
| --- | --- | --- |
| LPs | 80% | Stays in the pool vault, increasing per-BPT value over time |
| Protocol | 20% | Collected separately by the protocol admin |

The LP share is implicit — fees accumulate inside the pool, which
increases the value of each BPT (pool token).

The protocol's share is configurable per pool (default 20%, max 50%).

---

## What an LP sees over time

1. Trader swaps Token A → Token B in a pool you've LP'd into.
2. A swap fee (e.g. 0.3% of the input) is taken at the start of the
   swap.
3. The LP portion of that fee stays in the pool; the protocol portion
   is tracked separately for collection by the treasury.
4. When you withdraw, you get your proportional share of the pool's
   current balances — which include all the accumulated LP fees.

You never need to claim fees explicitly. Holding BPT is enough.

---

## Properties

- **All LPs earn proportionally.** Unlike concentrated liquidity AMMs
  (e.g. Uniswap V3), Cube does not require you to manage price ranges
  or tick positions. Every BPT holder earns at the same rate.
- **Fees compound automatically.** Accumulated fees increase the
  pool's base, so each subsequent swap generates fees on a larger
  amount.
- **IL still applies.** Like every AMM, Cube pools are subject to
  impermanent loss when token prices diverge from the ratio at the
  time of your deposit.

---

## Fee configuration

| Parameter | Range | Notes |
| --- | --- | --- |
| Swap fee rate | `0` – `100_000` hundredths of a bp (= 10%) | Typical pools: `1_000`–`10_000` (0.1%–1%) |
| Protocol fee rate | `0` – `5_000` bp (= 50%) | Default `2_000` (20%) |

---

## APY shown on the frontend

The Cube frontend and the DefiLlama yield-server adapter compute APY
from the trailing-24h LP fee income relative to TVL, annualized.
Actual returns depend on future volume.
