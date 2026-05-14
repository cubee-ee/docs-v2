# Pool Parameters

Each Cube pool is configured with a set of immutable and mutable
parameters. Integrators reading on-chain pool state need only the
fields below; the full account schema is in the program IDL.

---

## Immutable (fixed at pool creation)

| Parameter | Description |
| --- | --- |
| **Token list** | 2–10 unique SPL Token or Token-2022 mints. Each token's program is stored on the pool. |
| **Banned Token-2022 extensions** | Mints with these extensions are rejected at pool creation (`TransferFeeConfig`, `ConfidentialTransferMint`, `NonTransferable`, `InterestBearingConfig`, `PermanentDelegate`, `TransferHook` by default). |
| **Weights** | Basis points; sum = 10,000. Each weight in `[100, 9_900]` (1%–99%). |
| **Initial virtual balances** | One `u64` per token. Must be `> 0` and `≤ u64::MAX` (`18_446_744_073_709_551_615`). The pricing curve uses these; they grow on swaps (`checked_add`) and `MathOverflow` on u64 wrap. |
| **Leverage (UI only)** | UI-side multiplier feeding `initial_virtual_balance = floor(actual_amount × leverage)` via 1e9 fixed-point. UI bounds: `0.00001 .. 100_000_000` (100M). At init the hard ceiling is `floor(u64::MAX / amount_raw)` — e.g. ~1.84×10⁸ for a 100B-raw slot; the 100M cap keeps a ~2× buffer for swap-driven growth. |
| **Pool ID** | `u64` salt. PDA: `seeds = ["cubic_pool", config, pool_id.to_le_bytes()]`. |

---

## Mutable

| Parameter | Range / values | Changed by |
| --- | --- | --- |
| **Swap fee rate** | `u32`, hundredths of a basis point. Max `100_000` (= 10%). E.g. `3_000` = 0.3%, `10_000` = 1%. | Pool admin |
| **Protocol fee rate** | `u16`, basis points. Max `5_000` (= 50%). Default `2_000` (= 20%) — the share of each swap fee that goes to the protocol. | Protocol admin |
| **Pool enabled** | Bool. When `false`, all operations are blocked. | Protocol admin |
| **Swaps enabled** | Bool. When `false`, swaps are blocked but LPs can still add/remove. | Pool admin or protocol admin |
| **Banned extensions** | `u64` bitmap (config-level, affects new pools). | Protocol admin |

---

## Authority model (high-level)

Pools share a **CubicPoolConfig** account that carries two authorities:

- `pool_admin` — adjusts swap fee and the swap toggle
- `protocol_admin` — collects protocol fees, sets protocol fee rate,
  enables/disables pools, manages banned extensions

In production both are set to the protocol Treasury PDA, so every
admin operation goes through the treasury — see
[Pool Controls](../safety/pool-controls.md).
