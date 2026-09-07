# Cube XP

Cube XP is Coffer's **off-chain rewards accounting** for trading and liquidity participation. It is maintained by the backend, separately from pool reserves, BPT, and on-chain swap fees. Earning XP does not require staking BPT in an additional contract. An XP balance is not an SPL token balance or a specified token-conversion entitlement.

This page describes the implemented logic in local `backend-v2` branch `v5.1`, revision `a886497`. Runtime availability and current totals depend on the backend deployment and its indexers. Use the epoch and leaderboard endpoints for current displayed state instead of hard-coding “the current epoch” in a client.

## How points accrue

The backend runs an accrual job every three hours. Both swap XP and LP XP are processed by that job; neither is promised to appear immediately after a transaction.

| Activity | Epoch-one base rate | Measurement used by the checked implementation |
| --- | --- | --- |
| Swaps | 10,000 XP per USD of total pool swap fees | Indexed `feeAmountUsd` for the user's swaps in the preceding three hours |
| Liquidity | 6.25 XP per USD of LP value per accrual | The holder's current BPT share multiplied by the pool's USD TVL at the job |

The swap base is **total pool swap fee**, including the protocol share, rather than trade notional or LP-only fee. For example, a swap with $1 of indexed total pool fees earns 10,000 base XP in epoch one before a referral boost. This is an accounting example, not a return or profit estimate; executing the swap still incurs its fees and market effects.

For LP XP, the job discovers BPT holders through Helius DAS, reads each holder's share, and values it using the pool's recorded supply and TVL. It considers enabled pools with the current backend schema. It attempts to refresh metrics before accrual and can use cached values if that refresh fails. Holding transferred BPT can therefore count, subject to discovery and valuation, without requiring that the current holder made the original deposit.

The LP calculation is a sampled balance, not a continuous time-weighted integral. A constant eligible $1 position over eight successful accruals would receive 50 base XP in epoch one. Changes in balances, prices, pool eligibility, or data availability change the actual result. Withdrawing changes future eligible value; the accrual calculation does not deduct points already credited simply because liquidity was later removed.

## Epoch schedule

The checked implementation starts epoch one at **2026-05-17 00:00:00 UTC**. Each epoch is exactly **122 days**, approximately four months, and the multiplier halves at every boundary:

| Epoch | Multiplier | Swap XP per fee USD | LP XP per USD per three-hour accrual |
| --- | --- | --- | --- |
| 1 | 1 | 10,000 | 6.25 |
| 2 | 0.5 | 5,000 | 3.125 |
| 3 | 0.25 | 2,500 | 1.5625 |

For epoch `e`, the multiplier is `1 / 2^(e - 1)`. The accrual job uses the multiplier at its execution time. Epoch boundaries are not four calendar-month arithmetic, and a client should not substitute a fixed local countdown for the server's schedule.

`GET /api/leaderboard/epoch`, or `backend.getLeaderboardEpoch()`, returns the current epoch, start/next-start timestamps, milliseconds remaining, multiplier, base/current rates, and epoch history. The legacy response field `swapXpPerUsdLpFee` names the implemented **total-fee** rate despite the `LpFee` suffix. `lpXpPerUsd` is the rate for one three-hour accrual, not one day.

## Referrals

Referrals are implemented in this backend revision. An authenticated user can bind a referrer with `bindReferral(code, utm?)` and inspect their relationship and statistics with `getReferralStatus()` and `getMyReferrals(page, limit)`.

- A referred user receives a **3% boost** to their own swap and LP XP.
- The direct referrer receives a **12%** bonus based on that user's own XP for the accrual.
- The referrer's referrer receives a **6%** second-level bonus when the relationship and eligible accrual exist.

The bonus base already includes the referred user's own 3% boost. If a recipient of a referral bonus is also a referred user, their bonus is multiplied by 1.03 as well. The job applies `Math.floor` after multiplying each referral bonus base by its level rate and recipient boost. Referral bonuses are recorded separately from the user's own swap/LP accrual rows; do not recursively treat received bonuses as a new source of referral XP.

Binding requires the backend's authenticated eligibility checks; merely adding a code to a URL does not establish a relationship. Use the returned referral status as the account's recorded relationship.

## Reading XP in an integration

```ts
const epoch = await backend.getLeaderboardEpoch();
const ranking = await backend.getLeaderboard(1, 20);
const user = await backend.getLeaderboardUser(walletAddress);
const history = await backend.getLeaderboardUserHistory(walletAddress, 1, 50);
```

These methods return `SdkResult` values; check each result before reading its data. The public leaderboard exposes ranks and total points. User history exposes accrual time, swap XP, LP value/XP, and the own-XP total for each tick. An unknown leaderboard user can return 404 until the backend has a record.

Some history field names are retained for compatibility: `swapVolumeUsd` and `lastAccrualSwapUsd` contain the swap **fee-USD base** in the checked accrual code. Do not label those fields as total trading volume. Referral bonuses may make a user's cumulative total differ from a sum of their own swap/LP history rows.

Campaign methods also exist in the SDK, but their corresponding routes are absent from checked backend revision `a886497`. Their presence in TypeScript does not establish a currently active campaign, a prize pool, or payout terms. See the [REST availability table](../integration/api-reference.md#sdk-routes-not-present-in-the-checked-backend-revision).

XP depends on successful indexing, price inputs, holder discovery, and scheduled processing. Display the last accrual time and distinguish pending estimates from credited points. Pool fees and LP redemption remain governed by the [smart contracts](../technical/smart-contracts.md); XP does not change their accounting.

Sources: [accrual and referral calculations](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/leaderboard/leaderboard-cron.service.ts),
[rates and epoch constants](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/config/constants.ts),
[epoch schedule calculation](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/leaderboard/epoch.util.ts).
