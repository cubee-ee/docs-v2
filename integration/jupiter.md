# Jupiter Aggregator integration

`jup-integration/` ships a Rust adapter that implements
`jupiter_amm_interface::Amm` for Cubic Pool. Once registered with
Jupiter's router, every Cube pool becomes a routable liquidity source
on Jupiter — the same path Orca, Raydium, Meteora, and others use.

## Status

| Item                                                   | State |
|--------------------------------------------------------|-------|
| Trait impl (`from_keyed_account`, `quote`, `update`, `clone_amm`, `get_swap_and_account_metas`) | ✓ |
| Quote-parity with on-chain swap math                   | ✓ delegates to `cubic_pool::math::CubicMath::calc_out_given_in` |
| Account layout matches `cubic_pool::instructions::swap::Swap` | ✓ |
| `cargo test` against the pinned adapter versions       | ✓ |
| PR to `jup-ag/jupiter-amm-implementation`              | pending |

## Files

```
jup-integration/
  Cargo.toml          — workspace deps (cubic-pool, jupiter-amm-interface)
  src/
    lib.rs            — barrel
    cubic_amm.rs      — Amm trait impl
    math.rs           — apply_swap_fee + lp_balances + quote helpers
    state.rs          — internal CubicPoolState wrapper
  README.md
```

## Quote parity

`math::quote_amount_out` calls `cubic_pool::math::CubicMath::calc_out_given_in`
directly — the same compiled function the on-chain swap uses. Inputs
are pre-processed through `math::lp_balances` to strip
`protocol_fees_owed` (matches the on-chain handler exactly). For
identical pool state, the quote and the on-chain `amount_out` agree to
the ULP.

## Build and test

The adapter builds and tests against the pinned workspace dependencies:

```bash
cd jup-integration
cargo test
```

The quote path delegates to `cubic-pool` math directly, so dependency updates
should always be verified with the quote-parity tests before submitting an
upstream Jupiter PR.

## Submitting to Jupiter

1. Fork `jup-ag/jupiter-amm-implementation`. Add a `Swap::Cube` variant
   (or use the existing generic `TokenSwap`) and register
   `CubicAmm::from_keyed_account` in the loader.
2. Open a PR. Reach out to Jupiter integrations on Discord with a
   pointer to this doc and `jup-integration/README.md`.
