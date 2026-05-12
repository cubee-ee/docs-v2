# FAQ

## General

### What is a Cube pool?

A Cube pool is a multi-token automated market maker (AMM) on Solana with customizable weights and virtual liquidity. It uses the weighted constant-product formula (same mathematical foundation as Balancer) but with virtual balance amplification for capital-efficient pricing.

### How is Cube different from a standard weighted pool?

Standard weighted pools (like Balancer V1) use only actual deposited balances for pricing. Cube adds a **virtual balance layer** — the pricing formula operates on amplified balances that can be much larger than the actual tokens in the pool. This means tighter spreads and less price impact for the same amount of deposited capital.

### What chains does Cube support?

Cube is built natively on **Solana** using the Anchor framework. The smart contracts are written in Rust and compile to Solana BPF bytecode.

### Is Cube audited?

Cube has gone through professional security reviews. Audit reports will be published once the protocol moves out of its initial mainnet phase.

---

## Pool Creation

### How many tokens can a pool have?

Between **2 and 10** tokens per pool.

### What's the minimum weight per token?

**1%** (100 basis points). Maximum is **99%** (9,900 basis points). Weights must sum to exactly 100%.

### Can I change the weights after creating a pool?

No. Weights are **immutable** — they are set at pool creation and cannot be modified.

### Can I change the virtual balances after creating a pool?

Not directly. Virtual balances evolve naturally through trading, add/remove liquidity operations, and protocol fee collection. The leverage ratio (virtual/actual) is maintained proportionally through these operations.

### What token standards are supported?

Both **SPL Token** and **Token-2022**. Each pool stores a token program per
mint, so mixed SPL/Token-2022 pools are supported as long as every mint passes
the banned-extension checks. Certain Token-2022 extensions are banned by default
(TransferFeeConfig, ConfidentialTransferMint, NonTransferable,
InterestBearingConfig, PermanentDelegate, TransferHook).

---

## For Traders

### What types of swaps are supported?

Currently only **EXACT_IN** — you specify the input amount and receive a computed output. EXACT_OUT is not implemented; would require an on-chain `calc_in_given_out` (currently only TS off-chain quote).

### How are swap fees calculated?

Fees are charged on the input token:

```
fee = floor(amountIn * swapFeeRate / 1,000,000)
```

Where `swapFeeRate` ranges from 0 (no swap fee) to 10,000 (1%).

### What happens if my swap is too large?

Very large swaps relative to pool depth will experience significant price impact. The contract checks output against the LP-accessible balance (`actual_balance - protocol_fees_owed`) and reverts with `AmountOutExceedsBalance` if the quote would withdraw more than the pool can serve. The backend router automatically splits large swaps across multiple pools when available and skips routes that would fail this guard.

### Does Cube support multi-hop swaps?

Not natively. The current implementation supports direct swaps within a single pool. However, the backend router splits across multiple pools containing the same token pair for optimal execution.

---

## For Liquidity Providers

### How do I earn fees?

Swap fees accumulate in the pool's actual balances automatically. When you withdraw, you receive your proportional share of the (larger) actual balances. No claiming or harvesting is needed.

### What is BPT?

BPT (Balancer Pool Token) is the LP token minted when you deposit liquidity. It represents your proportional ownership of the pool's actual balances. Each pool has a unique BPT mint with 9 decimals.

### Can I deposit just one token?

Yes. The default UI path uses the `single_token_liquidity` helper program:
you provide one pool token, the helper swaps internally as needed, then calls
the proportional `add_liquidity` instruction and forwards the BPT to you. The
transaction still has a non-zero minimum BPT guard, so bad quotes or stale pool
state revert atomically.

Power users can still use the proportional mode directly by supplying every
live pool token.

### Can I withdraw just one token?

No. Withdrawals are also **proportional** — you burn BPT and receive all pool tokens proportionally.

### What about impermanent loss?

Like all AMMs, Cube pools experience impermanent loss. The virtual balance leverage amplifies both fee income and IL. Higher leverage = more fee revenue but more IL exposure. This trade-off is set at pool creation and cannot be changed.

### Are there minimum deposit amounts?

For the first deposit: the resulting BPT must be >= 1,000 raw units (0.000001 BPT with 9 decimals). This prevents griefing attacks that could brick the pool.

For subsequent proportional deposits: all live token amounts must be > 0.
Single-token deposits must set a non-zero `minimum_bpt_amount`.

---

## Fees

### How is the fee split between LPs and protocol?

The `protocol_fee_rate` determines the protocol's share of swap fees. Default is 20% (2,000 basis points), meaning:
- **80% of swap fees** → stay in pool (benefit LPs)
- **20% of swap fees** → tracked as `protocol_fees_owed` (collected by protocol authority)

### What's the maximum protocol fee?

**50%** (5,000 basis points) of the swap fee. The protocol can never take more than half.

### Can fees be changed?

Yes. The `pool_admin` can change the swap fee rate, and the `protocol_admin` can change the protocol fee rate. Both are subject to their respective maximums.

---

## Integration

### How do I find all available pools for a token pair?

```
GET https://api.cubee.ee/api/pools/by-pair?tokenA=<mint>&tokenB=<mint>
```

### How do I get the best swap route?

```
GET https://api.cubee.ee/api/pools/swap-route?tokenIn=<mint>&tokenOut=<mint>&amountIn=<amount>
```

This returns the optimal split across pools with expected outputs, price impact, and vault addresses needed to build the transaction.

### How do I build a swap transaction from the route?

The route response includes `tokenInIndex`, `tokenOutIndex`, `vaultIn`, `vaultOut`, and `poolAddress` for each split. Use these to construct Anchor `swap` instructions. See [Swap Routing](integration/swap-routing.md) for a complete TypeScript example.

### Does the backend verify on-chain data?

Yes. The backend verifies the Cubic Pool program bytecode against a reference binary (cached for 10 minutes). Pool data is fetched from on-chain during registration and validated. Swap routing always uses fresh on-chain balances.

---

## Technical

### What math precision does Cube use?

All on-chain calculations use fixed-point arithmetic with conservative rounding (the protocol never overpays the user). See [Pricing Model](technical/math.md).

### How are vaults derived?

Token vaults are **Associated Token Accounts (ATAs)** derived as: `ATA(pool_pda, token_program, mint)`. They are not stored on-chain — they're derived at runtime from the pool's public key, token program, and mint address.

### What's the account size for a pool?

**1,154 bytes** (8-byte Anchor discriminator + 1,146-byte data). This includes
per-token program IDs, protocol-fee balances, and 128 reserved bytes for future
extensions.

### How often does the backend sync with on-chain?

- **Balances**: Every 30 seconds (transaction parser) + every 10 minutes (metrics cron)
- **Swap routing**: Fresh on-chain fetch per request (10-second timeout)
- **Pool startup**: Full balance sync on backend restart
