# How LP fee income is generated

LPs earn the LP portion of a pool's **base input-token swap fee** through their
BPT share of actual reserves. There is no on-chain staking or harvest step for
these fees. A separate backend [XP program](../rewards/cube-xp.md) is not part of
this reserve accounting.

## Base fee and protocol share

For raw input `X`, `swap_fee_rate = f` and `protocol_fee_rate = p`:

```text
baseFee       = ceil(X * f / 1,000,000)
protocolBase  = ceil(baseFee * p / 10,000)
lpFee         = baseFee - protocolBase
curveInput    = X - baseFee
actualInDelta = X - protocolBase
```

`f` is in hundredths of a basis point, up to 100,000 (10%). `p` is in basis
points, up to 5,000 (50% of the base fee). The current pool's stored rate is what
applies; a config's default only initializes a new pool's protocol rate. Do not
assume every pool has a 20% protocol share.

With 6-decimal input, an input of 10 tokens, a 0.3% base fee (`f = 3000`) and a
20% protocol rate (`p = 2000`) produces 0.03 input-token base fee: 0.006 for the
protocol and 0.024 for LP reserves. Integer ceilings can change the effective
percentage on tiny amounts; a one-raw-unit fee can be entirely assigned to the
protocol even when the configured share is below 100%.

## Dynamic/surge fee has a different recipient

When enabled, the selloff policy also charges a surge fee in the **output**
token. The gross curve output leaves the LP-owned output balance; the trader
receives gross output minus surge, and **100% of surge is credited to the
protocol-fee bucket**. The base-fee protocol-share maximum does not cap this
separate charge.

Do not add input-token fee amounts directly to output-token surge amounts.
Convert each amount using its own token decimals and valuation if showing a
combined USD charge. The [dynamic-fee page](../for-traders/dynamic-fee.md)
explains the curve and four-segment calculation.

## LP balances and fee collection

`actual_balance` is already the LP-owned balance. `protocol_fees_owed` tracks
uncollected protocol base fees and surge fees separately. Protocol fee collection
sweeps those counters without reducing LP actual reserves or virtual balances.

Holding BPT gives a proportional reserve claim. On withdrawal, the LP receives
a share of the current basket, including accumulated LP base fees, subject to
rounding and the minimum BPT supply. This is not a guaranteed increase in USD
value: trading, external prices, inventory depletion, token behavior and
range-manager changes can offset fee income.

Fees remain inside LP reserves automatically; this does not mean every future
swap has larger volume or generates more fees. Impermanent loss and other
inventory losses can exceed earned fees. See [pool controls](../safety/pool-controls.md)
and [liquidity rounding limits](liquidity.md#rounding-limit-in-this-contract-version).

## Backend fields named APY

The checked backend revision `a886497` computes a simple annualized fee-income
rate, although its response fields are named `apy`, `apy24h` and `apy7d`:

```text
apy = apy24h = (LP fee income over 24h / current TVL) * 365 * 100
apy7d       = (LP fee income over 7d / current TVL) * (365 / 7) * 100
```

These are percentage values, not compounded `(1 + dailyYield)^365 - 1` values.
The backend returns zero when the relevant TVL or fee income is nonpositive.
The calculation uses indexed LP-fee buckets and external token prices; it is
not an on-chain promise and may lag recent activity. It is also not a projection
of a holder's total return or a reason to count protocol surge income as LP yield.

Sources: [swap handler](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/swap.rs),
[fee collection](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/admin/collect_protocol_fees.rs),
[backend analytics](https://github.com/coffer-so/backend-v2/blob/a8864979a70b73d266a5ee0b88e9145e88354974/src/analytics/analytics-cron.service.ts).
