# Core concepts

## Pool, config and token set

A pool belongs to one `CubicPoolConfig` governance account and contains 2–10
ordered, distinct token mints. The token set and each mint's token program are
chosen at initialization. Indices in quotes, instructions, balances and events
refer to this stored order, not symbol order or a wallet's token list.

Weights are stored as basis points: each is 100–9,900 and their sum is 10,000.
They affect both the marginal exchange rate and the curvature of the weighted
swap formula. They are **not necessarily immutable**: an enabled range manager
can update virtual balances and weights using compare-and-swap values, timing
limits, per-update change limits and optional leverage bounds. See
[pool controls](../safety/pool-controls.md).

## Three balances with different meanings

For each token the pool stores:

| Field | Meaning |
| --- | --- |
| `actual_balance` | LP-owned reserve used for deposits, withdrawals and the output availability check |
| `virtual_balance` | Balance used by the pricing curve; not an independently redeemable asset |
| `protocol_fees_owed` | Protocol-owned tokens awaiting collection, including output-token surge charges |

Normal supported settlement maintains the accounting relation
`vault amount = actual_balance + protocol_fees_owed`. Direct external donations
can leave unaccounted vault excess, so this relation is an accounting target, not
proof that every unsolicited transfer has been incorporated into LP balances.
**Do not subtract protocol fees from `actual_balance` a second time.**

A swap prices its fee-adjusted input against virtual balances. Its gross curve
output must fit inside the actual output reserve. Part of that gross output may
be retained as a surge fee instead of going to the trader. The complete fee and
balance changes are in [pricing math](../technical/math.md).

## Virtual depth and leverage

For a live token, leverage is `virtual_balance / actual_balance`.
`CubicPoolClient.sync()` also exposes `concentration`, which is the inverse,
`actual_balance / virtual_balance`. Neither figure is a loan balance or a claim
on tokens outside the vault.

Changing virtual balances changes the curve's price/depth without depositing
those virtual tokens. Swaps change both actual and virtual balances. Proportional
LP operations scale virtual balances and selloff-window state using the join or
burn ratio; integer rounding can prevent exact ratio preservation. Protocol fee
collection removes the separate fee bucket without reducing LP actual balances.

## A zero-balance token is different from a disabled token

A token with `actual_balance == 0` is called sidelined on these pages. It has no
LP output reserve, so a positive output cannot be paid from that slot. A normal
proportional deposit must also offer zero for that slot; it cannot reactivate a
zero balance at a depositor-chosen price. A successful swap using that token as
**input** can make its actual balance positive again.

`is_active` is a separate input-swap gate. It can block a token from being sold
into the pool even when the token has reserves. Turning it off does not by itself
remove the token from a proportional liquidity basket or prevent buying it.

The current pool admin can seed only some tokens, provided the first deposit includes at
least one positive amount. The seed is restricted to the current pool admin.
The single-token helper requires its selected input token to have a positive
actual balance. See [liquidity](../for-lps/liquidity.md).

## BPT: the pool's liquidity share

BPT is minted on a deposit and burned on withdrawal. Its mint is the PDA
`[b"bpt_mint", pool_pubkey]` under the cubic-pool program, has 9 decimals, and is
controlled by the pool PDA. Classic SPL Token is the default BPT program;
Token-2022 BPT is also supported and the SDK reads the mint's actual owner.

A holder's proportional claim is based on LP actual reserves and BPT supply,
subject to integer rounding and the minimum remaining supply. BPT does not give
its holder protocol-admin or pool-admin powers. A withdrawal leaves at least
1,000 raw BPT in circulation; these are not a separately minted burn-address
allocation. They remain in wallets because the effective burn is capped.

LPs receive their share of the input base fee through pool reserves. The output
surge fee belongs entirely to the protocol. Fee income, token prices, trades,
range-manager changes and rounding all affect the value of a BPT position.

## Snapshot-based quotes

Call `sync()` before using `CubicPoolClient` quotes. The snapshot includes the
full v5 state and the Solana Clock timestamp. Swap and single-token quotes can
also take an anticipated execution timestamp. A quote is a computation on that
state, not a reservation of inventory or selloff capacity.

The [SDK guide](../sdk/index.md) covers quote/build methods; the
[account reference](../technical/accounts-events.md) describes the stored fields.
