# FAQ

## What does Coffer do?

It exchanges tokens within weighted multi-token pools on Solana. Pools use
virtual balances for pricing and actual balances for settlement. BPT holders
share the LP-owned reserves. The implementation and SDK retain some Cube names.

## How many tokens can a pool contain?

The contracts and SDK accept 2–10 distinct mints. The transaction must still
fit Solana's wire size and execution limits. A historical 9-token UI limit is
not a 9-token SDK or contract rule. Larger operations use ALTs; single-token
deposits may need separate account setup.

## Are weights and virtual balances immutable?

No. Weights start at 1%–99% each and sum to 100%, but an enabled range manager
can change weights and virtual balances within its configured permissions.
Updates include expected-current values to detect stale state. Virtual balances
also change through swaps and proportional liquidity operations. See
[Pool controls](safety/pool-controls.md).

## What changes when the pool admin is disabled?

The pool-admin key is cleared, blocking its privileged operations. This does
not automatically disable an already appointed range manager, remove protocol
governance or remove program upgrade authority. Renouncing one role does not
make the whole protocol immutable.

## Which tokens are supported?

Classic SPL Token and compatible Token-2022 mints can share a pool. BPT can
also use either program. Creation policy and runtime transfer support are
different checks: passing a mint's creation policy does not make transfer fees,
hooks or every issuer-controlled extension safe or supported. The SDK rejects
known incompatible transfer paths. See [Pool parameters](overview/pool-parameters.md).

## What kinds of swaps exist?

The contract has exact-input swaps: specify input and a minimum output.
It does not expose an exact-output swap instruction. A router can compose
multiple pool instructions, but that is not a new on-chain swap mode.

## How are the fees charged?

The base fee is `ceil(amountIn * swapFeeRate / 1,000,000)` in the input token.
The maximum base rate is 100,000, or 10%. The protocol's base-fee share is also
rounded upward: `ceil(baseFee * protocolFeeRate / 10,000)`, with rate at most
5,000, or 50%.

An enabled selloff policy can add a **surge fee in the output token**, all of
which goes to the protocol. The 50% limit concerns the base fee's configured
split, not the separate surge charge. Read [Dynamic fee](for-traders/dynamic-fee.md).

## Is the dynamic fee just the final window percentage times output?

No. The implementation averages a piecewise-linear rate curve over portions
of the input-token window and charges four portions of the taxed span using
actual curve-output differences. The threshold, low/mid/high values, kink,
integer rounding and earlier swaps in the transaction affect the result.
It is not an exponential curve or a single final-utilization multiplier.

## When does max selloff reset?

It is a sliding window with current usage plus a time-decayed previous bucket,
not a daily wallet allowance. The cap uses a virtual-balance snapshot and is
shared by everyone selling that input token into the pool. A window change,
long idle period, or liquidity rescaling updates the stored basis differently.
See the complete [window rules and examples](for-traders/max-selloff.md).

## What can make a swap fail?

Disabled pool/swaps, an inactive input token, exceeded selloff capacity,
insufficient actual output inventory, minimum-output failure, integer bounds,
authority/account mismatch or token-program restrictions. Gross curve output
is checked against `actual_balance`; protocol fees must not be subtracted again.
A successful earlier quote does not reserve inventory or window capacity.

## Can I deposit only one token?

Yes, for a seeded pool with a positive actual balance of the selected input.
The deployed helper performs swaps and a proportional join, then forwards BPT
and refunds remaining tokens. The whole deposit has a final BPT minimum.
See [Single-token deposit](sdk/single-token-deposit.md).

## Must the first deposit contain every token?

No. Only the current nonzero pool admin can seed; at least one amount must be
positive. Other tokens may start with zero actual balance. Initial BPT comes
from the virtual-balance invariant and must be at least 1,000 raw BPT.

## Is an oversized token in a proportional deposit donated?

No. Supplied amounts are spend ceilings. The contract crops them to the
limiting proportional ratio and leaves the unused portion in the wallet.
The SDK quote reports both the transferred and unused amounts.

## Can I withdraw everything or choose only one output token?

Removal pays proportional actual reserves. It caps the BPT burn to leave
1,000 raw BPT in circulation; unused BPT stays in the wallet. There is no direct
single-token withdrawal. Use `quoteRemove().data.effectiveBptIn` and its output
vector when setting withdrawal minimums.

## Who earns the fees, and can an LP lose money?

LP reserves receive the base fee less its protocol share. The output surge
fee belongs entirely to the protocol. Fees are not a guaranteed total return:
market prices, inventory changes, token behavior and manager actions can reduce
a position's value. The current contract also has a small-amount deposit
[rounding limitation](for-lps/liquidity.md#rounding-limit-in-this-contract-version)
that the SDK conservatively guards against. A client guard is not a contract fix.

## Are all methods in the SDK available on every backend?

No. The SDK exposes the full three-program instruction ABI independently of
the REST server. Some newer `CubeBackendClient` methods target routes absent
from the checked local backend revision. See [API compatibility](integration/api-reference.md).
This source comparison does not establish which revision a live server runs.

## Do these pages prove that the deployed contracts are fully safe?

No. They document the selected implementation and its limits. ABI parity,
math reference tests and read-only integration checks are different from a
security audit or executing every administrative path on a validator. Use the
[version notes](technical/versions.md) to identify exactly what was checked.

## What is the current pool account size?

1,683 bytes including the Anchor discriminator. V5 uses fields in space that
was reserved in v4. Legacy v3 is 1,154 bytes and is not supported by the current
SDK parser or the deployed `migrate_to_v5` path. See [Accounts and events](technical/accounts-events.md).

## Is the SDK licensed the same way as the contracts?

The checked SDK is MIT. The checked contracts and documentation have BUSL-1.1
license files. Refer to the license of each repository and revision; see
[License](license.md).
