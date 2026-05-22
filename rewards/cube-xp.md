# Cube XP

**Cube XP** is the rewards program for liquidity providers and traders
on Cube. Every dollar swapped and every dollar of active liquidity earns
XP that accumulates on your wallet and is tracked on a public
leaderboard.

There will never be a better time to farm XP than the current epoch —
rewards halve at the end of every 4-month period.

---

## How XP is earned

| Action | XP rate | When it's credited |
| --- | --- | --- |
| **Swap** | **100 XP per $1** of swap volume *(flat rate today; will become pool-dependent — see below)* | Instantly, on every swap |
| **Provide liquidity** | **50 XP per $1 per day** | While the LP position remains active |

- **Swap XP** is one-shot — every dollar of input volume credits 100 XP
  at the moment the swap settles. There is no claim step. *Today this
  rate is flat across every pool; in a future release it will depend
  on the specific pool the swap routes through — see [Pool-dependent
  swap XP](#pool-dependent-swap-xp-coming-soon) below.*
- **Liquidity XP** is continuous — XP accrues for as long as your BPT
  (pool token) balance is non-zero. Withdrawing liquidity stops accrual
  on the withdrawn portion immediately.

> Under the hood, liquidity XP is credited by a worker every 3 hours
> (≈ 6.25 XP per $1 per 3h). The UI rounds this to a clean **50 XP / day**
> headline number — the on-chain economics are identical.

XP appears in two places in the app:

- The **swap confirmation modal** shows the XP you'll earn for that trade.
- The **add liquidity** flow shows the daily XP rate your deposit will
  start accruing.

---

## Pool-dependent swap XP (coming soon)

The current **100 XP per $1 of swap volume** is a flat, protocol-wide
rate that applies to every pool equally. This is intentionally simple
for the launch of the program, but it is **not the long-term design**.

In a future release, swap XP will become **pool-dependent** — i.e. the
exact number of XP credited per $1 of swap volume will vary from pool
to pool. The reason is straightforward: different pools have different
swap fee rates and therefore generate different amounts of revenue for
the protocol per dollar of volume routed through them. A $1 swap
through a high-fee, low-liquidity pool is worth meaningfully more to
the protocol than a $1 swap through a tight, low-fee pool, and the XP
rate will eventually reflect that. Concretely, expect higher-fee pools
and pools that the protocol wants to incentivise (e.g. newer pairs,
strategically important pairs) to credit **more** XP per $1, while
very tight low-fee pools may credit **less**. The exact multipliers
will be published per pool and surfaced in the swap UI before the
change ships.

**What this means for you today:**

- Until the change is rolled out, every pool credits the same flat
  **100 XP per $1** (subject to the current epoch's halving — see
  below).
- After the change, the XP figure shown in the swap modal will already
  reflect the per-pool rate, so you don't need to track multipliers
  manually. No retroactive adjustments will be applied to XP earned
  before the change.

---

## Epochs and halvings

Cube XP runs in **epochs**. Each epoch lasts **4 months**, and at the
end of every epoch the per-dollar XP rates are **cut in half**.

| Epoch | Swap rate (base) | LP rate (per $1 per day) |
| --- | --- | --- |
| 1 (current) | 100 XP / $1 | 50 XP / $1 |
| 2 | 50 XP / $1 | 25 XP / $1 |
| 3 | 25 XP / $1 | 12.5 XP / $1 |
| … | halved each epoch | halved each epoch |

The swap rate column above is the **base / protocol-wide rate** —
once pool-dependent swap XP ships, each pool's effective rate will be
this base rate multiplied by that pool's individual multiplier. The
halving schedule still applies on top: at the start of each new epoch
the base rate is halved, and every pool's effective rate halves with
it.

We are currently in **epoch 1** — the most rewarding epoch the program
will ever offer. Every subsequent epoch issues half as much XP per
dollar.

A **countdown to the next epoch** is shown in the interface so you
always know how long the current rate lasts before the next halving.

XP already earned is **not affected by halvings** — only the rate at
which new XP is generated changes.

---

## Leaderboard

A public leaderboard ranks wallets by total XP earned across all
epochs.

- LPs and traders compete on the same board — both activities convert
  into the same XP unit.
- Rankings update as new XP is credited (swaps in real time, LP
  positions every 3h).
- Early participation in epoch 1 has the largest impact on long-term
  standing, because later epochs accrue XP more slowly.

The goal is not just a leaderboard for its own sake but a **complete
long-term rewards system** for Cube users.

---

## What XP is for

XP is the protocol's single, persistent record of contribution to Cube.
We expect the program to run for roughly **12–18 months** of decaying
emissions across multiple epochs.

If Cube ever introduces a token in the future, XP **may eventually be
convertible** into it. This is not a commitment to a token launch —
just a statement that XP is intended to be the basis on which any
future allocation to LPs and traders would be calculated.

---

## Tips for farming XP

- **Start early in the epoch.** Day 1 of epoch 1 is the highest XP-per-$
  rate the program will ever pay.
- **LP XP compounds with time.** Because LP XP accrues per day, leaving
  liquidity in for a full epoch is dramatically better than rotating in
  and out.
- **Trade size matters linearly.** Both swap and LP XP are exactly
  proportional to dollar size — no thresholds, no per-day bonuses.
  (Per-pool multipliers will be introduced for swap XP in a future
  release — see [Pool-dependent swap
  XP](#pool-dependent-swap-xp-coming-soon).)
- **Watch the countdown.** When the epoch timer is close to zero,
  consider front-loading activity before the halving.

---

## Links

- **XP Documentation** → this page
- **Leaderboard** → https://cubee.ee/leaderboard

---

## FAQ

**Do I need to claim XP?**
No. XP is credited automatically — instantly for swaps, every 3 hours
for active LP positions.

**Do I lose XP if I withdraw liquidity?**
No. XP already earned is permanent. Withdrawing only stops further
accrual on the withdrawn portion.

**Does XP earned in epoch 1 get halved when epoch 2 starts?**
No. Halvings apply to the **rate** at which new XP is earned, not to
balances already accrued.

**Is the LP XP really 50 / $ / day, or 6.25 / $ / 3h?**
They're the same number. The worker credits ≈ 6.25 XP per $1 every 3
hours; over 24 hours this equals 50 XP per $1. The UI shows the
daily number for simplicity.

**Will there be a referral program?**
A referral system is planned as a separate launch — including XP for
inviting friends and, longer-term, potential revenue-share in USD on
referred LP/trader activity. Details will be published in a dedicated
announcement.

**Is XP a token?**
No. XP is an off-chain accounting unit tracked by the Cube backend. It
is not transferable and has no on-chain representation today.

**Why will swap XP per $1 differ between pools later?**
Because different pools have different swap fee rates, and a dollar of
volume through a high-fee pool is worth meaningfully more revenue to
the protocol than a dollar through a tight low-fee pool. To keep XP
aligned with contribution to protocol revenue, swap XP per $1 will
become **pool-dependent** — see [Pool-dependent swap
XP](#pool-dependent-swap-xp-coming-soon). Until that change ships,
every pool credits the same flat 100 XP per $1 (subject to halvings).
