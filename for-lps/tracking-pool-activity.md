# Tracking pool activity

Use account reads for authoritative current state, confirmed transaction logs
for historical changes, and the backend for indexed/valued views. They have
different completeness and freshness guarantees.

## Read the full v5 state

```typescript
import { PublicKey } from "@solana/web3.js";
import { CubicPoolClient, getConfig } from "@cubee_ee/sdk";

async function readPool(poolAddress: PublicKey) {
  const client = new CubicPoolClient({ config: getConfig("mainnet"), poolAddress });
  const result = await client.sync();
  if (!result.ok) throw new Error(result.error.humanMessage);
  return result.data;
}
```

`sync()` verifies the pool owner, decodes its state, reads the BPT mint and
reserve-token mints, and includes the Solana Clock timestamp. The result exposes
balances, weights, token activation, all seven selloff-policy values, both
window buckets, the window timestamp/snapshot, pool admin, range-manager state,
leverage limits, banned-extension snapshot, ALT, BPT supply and token programs.
The pool read and subsequent mint/Clock reads are separate RPC requests; do not
assume all returned accounts were read at one atomic slot.

The pool layout is **1,683 bytes**, including its Anchor discriminator.
V5 reuses reserved space in the previous 1,683-byte layout. SDK decoding rejects
legacy 1,154-byte v3 accounts; `migrate_to_v5` is not a v3 reallocation path.
Complete field offsets/types are in [Accounts and events](../technical/accounts-events.md).

For a lower-level reader, `decodePoolAccount` exposes the SDK's camelCase raw
fields, while `decodeContractAccount("cubicPool", "CubicPool", data)` preserves
the IDL names and fixed arrays. Verify ownership before using a standalone
parser. Slots beyond `token_count` are not live pool tokens.

## `get_pool_info` is a partial event view

This read-only instruction emits a `PoolInfo` event and can be simulated. It
reports composition, derived vaults/BPT mint, balances, fees, enabled flags and
timestamp. It does **not** include every newer account field: for example,
selloff policy/window state, banned extensions, admin/range-manager settings,
ALT and BPT total supply require account reads.

Its named account is just `pool`, with no instruction signer; simulation still
needs a valid transaction context. The emitted data is an event, not a typed
return value from this particular instruction.

## Indexing transaction events

The SDK exposes all 60 declared events across the three programs. Prefer
`parseContractEvents(logs)` for exact IDL field names and BN integers, or the
existing `parseCubicPoolEvents` compatibility API for camelCase event fields.

| Event | Relevant meaning |
| --- | --- |
| `Swap` | Input, net user output, input base/protocol fee, output surge fee, user/pool and timestamp |
| `LiquidityAdded` | Actual deposited amounts and BPT minted |
| `LiquidityRemoved` | Actual output amounts and effective BPT burned after the supply floor |
| `PoolStateLog` | Pool address, actual/virtual/fee vectors and timestamp |
| `MaxSelloffWindowAdvanced` | State emitted when a capped input swap advances the limiter successfully |
| `MaxSelloffSet` | Policy update; read the account for all fields, including mid slope/kink |
| Range-manager/admin events | Authority and configuration changes; fields differ by event |
| `SingleTokenDeposit` | Final helper-deposit summary alongside the internal swap/join logs |

`PoolStateLog` is emitted by swap, add and remove; it is not emitted by every
administrative action and is not a complete pool-account snapshot. It does not
carry BPT supply. Read the mint and pool accounts for fields absent from events.

Before applying indexed state:

1. Verify transaction success (`meta.err == null`) at the chosen commitment.
2. Attribute log payloads to the actual invoking program, including nested CPIs.
   Matching an event discriminator alone does not authenticate the emitter.
3. Preserve transaction/instruction ordering and make ingestion idempotent.
4. Treat a helper deposit as one user action with internal swap/join events;
   avoid double-counting its deposit, fees or BPT.
5. Reconcile gaps with account reads. Failed transactions may contain logs but
   their attempted pool changes did not commit.

## Balance and value checks

Normal supported settlement accounts for the vault as
`actual_balance + protocol_fees_owed`. Actual is already the LP portion.
Unexpected vault excess can be an external donation; it does not automatically
increase the recorded LP claim. A deficit or inconsistent token-program owner
requires investigation, not silently rewriting the SDK snapshot.

With external token prices `price[i]` in USD per whole token:

```text
LP TVL       = sum(actual[i] / 10^decimals[i] * price[i])
virtual TVL  = sum(virtual[i] / 10^decimals[i] * price[i])
position USD = rawUserBpt / rawBptSupply * LP TVL
```

Virtual TVL is pricing depth, not additional inventory. Position value is a
valuation of a share; actual withdrawal still applies integer rounding and the
minimum remaining BPT supply. Price-source coverage and freshness matter.

For an infinitesimal fee-free exchange, the spot quote in **whole output tokens
per one whole input token** is:

```text
outPerIn = (virtualOut * weightIn) / (virtualIn * weightOut)
           * 10^(decimalsIn - decimalsOut)
```

The reciprocal is input per output. Do not invert the label accidentally, or
use a raw balance ratio without weights/decimal conversion. Executable quotes
also include curve impact, base/surge fees and state checks.

For positive actual balances, `leverage = virtual / actual`; SDK `concentration`
is the inverse. A zero actual balance makes leverage undefined rather than
zero. Track these alongside weights, manager permissions, activation flags and
window capacity when evaluating a pool.

## Backend metrics

`CubeBackendClient` has methods for pool details, transaction history, transaction
counts, platform stats, time-series statistics and portfolio views. See the
[API reference](../integration/api-reference.md) for exact routes, response
envelopes and which methods the checked backend implements.

The backend's pricing, cache and indexing jobs are not atomic on-chain state.
Check response timestamps and missing values. Do not use an API TVL value as the
raw reserve input to SDK swap or liquidity math. Its fields named `apy` are
simple annualizations in the checked revision; see [fee income](how-yield-is-generated.md).

Sources: [event definitions](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/events.rs),
[get_pool_info](https://github.com/coffer-so/contracts/blob/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs/cubic-pool/src/instructions/user/get_pool_info.rs),
[SDK parsers](https://github.com/coffer-so/sdk/tree/27de819c469056bfb7cd3ab3a4cfdbde741db2f8/src/parsers).
