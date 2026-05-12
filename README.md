# What is Cube?

**Cube** is a weighted automated market maker (AMM) on **Solana**.
Each pool holds 2–10 tokens with configurable weights and a virtual
liquidity layer that lets pools quote tight spreads without
requiring deep on-chain inventory.

---

## At a glance

| Feature | Description |
| --- | --- |
| Multi-asset pools | 2–10 tokens per pool |
| Custom weights | 1%–99% per token, sums to 100% |
| Capital-efficient pricing | Virtual liquidity layer for tight spreads |
| Proportional liquidity | No tick management — all LPs earn proportionally |
| Configurable swap fee | Up to 10% per pool |
| SPL Token & Token-2022 | Both standards supported (with extension allow-list) |
| Smart order routing | Public backend splits swaps across pools for best execution |

---

## What you can do with the API

| You want to | Use |
| --- | --- |
| Get a quote for a swap | [`@cube/sdk` → `CubicPoolClient.getSwapQuote`](sdk/index.md) |
| Compose a swap transaction | [`@cube/sdk` → `CubicPoolClient.swap`](sdk/index.md) |
| Route a swap across multiple pools | [Swap Routing](integration/swap-routing.md) |
| List pools / find by token pair | [API Reference](integration/api-reference.md) |
| Index live trades | Subscribe to the `Swap` event on the on-chain program |
| Show TVL / volume / fees | Use the protocol's [DefiLlama listing](https://defillama.com/protocol/cube) |

---

## Quick links

- [Core Concepts](overview/core-concepts.md) — pool composition, weights, virtual liquidity
- [Pool Parameters](overview/pool-parameters.md) — what's configurable per pool
- [Pricing Model](technical/math.md) — swap formula and fee behaviour
- [Smart Contracts](technical/smart-contracts.md) — instructions integrators call
- [API Reference](integration/api-reference.md) — backend REST endpoints
- [Swap Routing](integration/swap-routing.md) — split-route execution
- [Jupiter Aggregator](integration/jupiter.md) — Jupiter integration
- [Pool Controls](safety/pool-controls.md) — pause flags and authority model
- [FAQ](faq.md)
- [License (BUSL-1.1)](license.md) — Cube is source-available, not open source

---

> **Note on licensing.** Cube is released under the **Business Source License 1.1**. The source is public so it can be reviewed and integrated against, but it is **not permission to fork and redeploy the protocol**. See [License](license.md) for details. The license auto-converts to Apache 2.0 on 2030-05-12.
