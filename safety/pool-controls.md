# Pool Controls

Cube protocols expose two pause flags per pool that integrators
should be aware of. Both are controlled by the protocol treasury,
not by individual wallets.

---

## Authority model

Every admin operation on a Cube pool requires a signature from the
protocol treasury (a PDA, not a wallet keypair). Pool / protocol
admin fields on a pool both resolve to this treasury, so there is
no path by which a single wallet can change pool parameters or
move funds.

Practical implications for integrators:

- Pool parameters (swap fee, pause flags) can only change via
  treasury-signed transactions — they cannot be silently flipped
  by a single keyholder.
- Treasury control itself can be rotated; rotation is two-step
  (initiate by current admin, accept by new admin) so it is
  observable on-chain before it takes effect.

---

## Pause flags

Each pool has two independent flags.

### `pool_enabled`

| Value | Effect |
| --- | --- |
| `true` (default) | All operations allowed |
| `false` | All operations blocked — swap, add liquidity, remove liquidity |

Use case: full pool shutdown — emergency, migration, decommission.

### `swaps_enabled`

| Value | Effect |
| --- | --- |
| `true` (default) | Swaps allowed |
| `false` | Swaps blocked, but LPs can still add and remove liquidity |

Use case: pause trading during an investigation while letting LPs exit.

### Control matrix

| Operation | `pool_enabled=true`, `swaps_enabled=true` | `pool_enabled=true`, `swaps_enabled=false` | `pool_enabled=false` |
| --- | --- | --- | --- |
| Swap | ✅ | ❌ | ❌ |
| Add liquidity | ✅ | ✅ | ❌ |
| Remove liquidity | ✅ | ✅ | ❌ |

Aggregators should treat a pool with `pool_enabled = false` or
`swaps_enabled = false` as **inactive** and skip it for routing.
Both fields are on the `CubicPool` account and refresh on every
sync.

---

## Emergency recovery

The treasury can withdraw funds from a pool in an emergency. This
path is gated by the treasury authority, requires the pool to be
disabled first, and is designed to be a one-way operation: a pool
that has gone through emergency recovery is retired and will remain
disabled permanently. Integrators do not need to handle this path
specially — a disabled pool already drops out of routing.
