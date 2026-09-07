# Coffer: the Cubic Pool AMM

Coffer is a weighted multi-token AMM on Solana. Pools keep actual token reserves
for settlement and virtual balances for pricing. Liquidity providers hold BPT,
the pool's share token; traders exchange one pool token for another.

These pages describe the `audit-fixes-excluded-SF` contract revision `96a2ee2`
and SDK `@cubee_ee/sdk` source version **0.11.1**, revision `09cc776`, checked on
**2026-09-07**. Some code identifiers still use the Cube name. See the
[version and compatibility notes](technical/versions.md) before integrating.

## Capabilities

| Capability | Current behavior |
| --- | --- |
| Pool composition | 2–10 distinct mints; SDK accepts 10, transaction size must still fit |
| Token weights | 1%–99% each, total 100%; an enabled range manager can change them within configured limits |
| Virtual liquidity | Pricing depth is separate from the actual reserves available to pay out |
| Liquidity | Seed by the current pool admin, proportional add/remove and a single-token deposit helper |
| Base swap fee | Input-token fee, configurable up to 10% |
| Dynamic fee | Optional output-token surge charge tied to the input token's sliding selloff window |
| Protocol accounting | Protocol fees are tracked separately from LP-owned actual balances |
| Token programs | Classic SPL Token and compatible Token-2022 mints, including Token-2022 BPT |
| Contract access | Three programs, 59 instructions; the SDK exposes the complete typed instruction ABI |

Virtual depth does not create redeemable inventory. A quote that exceeds the
available actual output balance fails, even when the virtual curve has capacity.
Fees and depth do not guarantee an LP return or a particular market price.

## Start here

- [Core concepts](overview/core-concepts.md): reserves, weights, BPT and token state.
- [Liquidity](for-lps/liquidity.md): seed, spend ceilings, withdrawal floors and rounding.
- [Swapping](for-traders/swapping.md): exact-input settlement and minimum output.
- [Dynamic fee](for-traders/dynamic-fee.md): threshold, slopes, kink, four-segment calculation and examples.
- [Max-selloff window](for-traders/max-selloff.md): snapshot cap, carryover and window changes.
- [Pool controls](safety/pool-controls.md): authority roles, pauses and range-manager powers.
- [SDK](sdk/index.md) and [SDK reference](sdk/reference.md): actual method names, arguments and return types.
- [Read state and calculate a swap](sdk/state-and-quotes.md): every contract-to-SDK field mapping, config/Treasury reads, quote inputs and slippage output.
- [Contract instruction reference](technical/instruction-reference.md): every on-chain instruction.
- [Accounts and events](technical/accounts-events.md): storage layouts and indexing fields.

## Read and quote through the SDK

```typescript
import BN from "bn.js";
import { PublicKey } from "@solana/web3.js";
import { CubicPoolClient, getConfig } from "@cubee_ee/sdk";

async function quoteOneToken(poolAddress: PublicKey) {
  const client = new CubicPoolClient({ config: getConfig("mainnet"), poolAddress });
  const state = await client.sync();
  if (!state.ok) throw new Error(state.error.humanMessage);

  const amountIn = new BN(10).pow(new BN(state.data.tokens[0].decimals));
  return client.quoteSwap(0, 1, amountIn, 1_000); // 0.1% slippage budget
}
```

This reads accounts and computes a quote; it does not sign or send a transaction.
Builders, wallet signing, sending and confirmation are separate steps. Use the
final minimum output or minimum BPT from a refreshed quote.

## Backend and licensing

The [backend API](integration/api-reference.md) provides indexed pool data,
portfolio information and routing. Its availability, authentication and update
cadence are separate from the contracts and SDK. Some SDK methods require a
newer backend than the source snapshot checked here; the API reference marks them.

Licenses differ by repository: the checked SDK is MIT; the checked contracts and
this documentation use BUSL-1.1. See [License](license.md) for the authoritative files.
