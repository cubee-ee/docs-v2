# Core Concepts

Three things you need to know to integrate with Cube pools.

---

## Pool tokens

Each pool holds **2–10 SPL tokens** (SPL Token or Token-2022). The set
of tokens, their weights, and the pool's swap-fee bounds are fixed at
pool creation.

Read pool composition from the on-chain `CubicPool` account
(its layout is published in the program IDL — see
[Smart Contracts](../technical/smart-contracts.md)).

---

## Weights

Each token has a **weight** (basis points, sum = 10,000). Weights are
**immutable**.

Weights determine the curvature of the swap formula: trading **into**
a high-weight token costs less slippage than trading into a low-weight
token. A 50/50 pool prices symmetrically; an 80/20 pool resists trades
that move the 80% token.

---

## Liquidity depth (virtual balances)

Cube pools price swaps against a **virtual liquidity** depth that can
exceed the tokens physically held in the pool. This is how a small
pool can quote a tight spread.

What an integrator needs:

- The on-chain `CubicPool` account exposes the balances the program
  uses for pricing. Pass them to the swap formula in
  [Pricing Model](../technical/math.md) and you get the same quote
  the program will produce.
- A swap can never pay out more than the **actual** tokens currently
  available — the program enforces this and reverts otherwise.
- Both `add_liquidity` and `remove_liquidity` are proportional across
  all tokens; the program adjusts pool state internally so subsequent
  quotes remain consistent.

If you only need quotes, prefer the SDK helper
([`CubicPoolClient.getSwapQuote`](../sdk/index.md)) or the public swap
router rather than re-implementing the math.

---

## BPT (pool token)

Liquidity providers receive **BPT** when they deposit. It's a standard
SPL token (decimals: 9) representing their proportional share of the
pool. BPT is burned when liquidity is withdrawn.

- Mint address: PDA `[BPT_MINT_SEED, pool_address]`
- Mint authority: the pool PDA itself

BPT value is denominated in the pool's actual balances and grows over
time as swap fees accumulate.
