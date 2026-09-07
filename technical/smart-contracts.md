# Smart Contracts

This reference describes contracts [`audit-fixes-excluded-SF` at `96a2ee2`](https://github.com/coffer-so/contracts/tree/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0/programs) and SDK [`0.11.1` at `09cc776`](https://github.com/coffer-so/sdk/tree/09cc7766a1e865b5c3f9b97a0526a982671683f5/src). Defaults below are values written by this code; an existing account may contain different settings.

## Programs and deployment

| Program | Mainnet address | Responsibility |
| --- | --- | --- |
| `cubic_pool` | `8iQtGj9mcUfFUGaiCpPy89swC3s8YTC8FhVZWfgeZhwu` | Pool creation, weighted swaps, BPT liquidity, per-pool controls and config governance |
| `protocol_admin` | `3jiojHZbjJQ7QLMGSTjFwxVEmx4NtuRy34nLAmsJME81` | Treasury, protocol authority, supervisor, program upgrades and administrative CPI wrappers |
| `single_token_liquidity` | `7BpdUH1tzTSXLuQNo6YpjJ8Eagw8AkrS6cnkxiJdCFS2` | Atomic single-token deposit through swaps and a proportional join |

All three were deployed and checked during the 2026-09-07 mainnet rollout. Their upgrade authority at that verification was Treasury PDA `B4gyhrqLzX36VEu54ShqgTkb4TpXooTdxtcmvER3d3Bg`, derived from `["treasury"]` under `protocol_admin`. Program addresses do not prove that the binary or authority is unchanged: re-read ProgramData when performing a later deployment review.

The interface has **59 instructions**: 28 in Cubic Pool, 29 in Protocol Admin, and 2 in Single Token Liquidity. The [instruction reference](instruction-reference.md) lists every argument, fixed account and required remaining-account layout. The [accounts and events reference](accounts-events.md) lists stored fields and all 60 event schemas. The SDK ships these three IDLs; it is not necessary to obtain them from a TVL adapter.

## Public operations

`initialize_cubic_pool` is permissionless under an existing config. Its payer becomes that pool's admin. It creates the pool account and BPT mint, with zero actual reserves; it does not create reserve vault ATAs or supply liquidity. The current pool admin prepares the needed token accounts and performs the seed deposit through `add_liquidity`; see [account setup](../for-lps/liquidity.md#account-setup). Config creation itself requires the Treasury's authorized CPI.

`swap` takes exact gross input and a minimum net output. It validates both token indices, mints, token programs, vault addresses and user accounts. Pool trading must be enabled and the input token must be active. It advances the input token's selloff window using gross input, calculates the base input fee, prices the post-base-fee input against virtual balances, and deducts any surge fee from output. Output cannot exceed the LP-owned actual reserve. The complete rounding and fee sequence is in [math](math.md), [swapping](../for-traders/swapping.md), and [max-selloff](../for-traders/max-selloff.md).

`add_liquidity` has two modes. At zero BPT supply, only the non-renounced pool admin may seed it; at least one input is nonzero, and initial BPT is calculated from virtual balances and weights. Later deposits are proportional: `token_amounts` are ceilings, excess stays in the wallet, and every live slot must have a positive ceiling. A zero-reserve slot takes zero and can become live through a later swap into that token. The current SDK rejects joins where a live token's actual transfer rounds to zero; this is a conservative client check and is not a replacement for contract enforcement. See [liquidity](../for-lps/liquidity.md).

`remove_liquidity` burns BPT and returns a proportional basket of LP-owned actual reserves. The effective burn is capped so supply remains at least 1,000 raw BPT (0.000001 BPT at 9 decimals). The unused portion of the requested burn stays in the user's account; minimum token outputs apply to the effective burn. LP operations require `pool_enabled`, but do not require `swaps_enabled` or each token's input-active flag.

`deposit_single_token` is a separate program, now available on mainnet. For an already seeded pool it receives one token, swaps allocated portions in pool order, joins using the resulting basket, returns newly minted BPT and refunds token dust. The final positive `minimum_bpt_amount` protects the whole atomic operation. Internal swaps use zero per-leg minimum and apply the same base fee, surge fee, selloff limiter and input-token gate as direct swaps. See [single-token deposits](../sdk/single-token-deposit.md).

## Accounting and token programs

For each token, `actual_balance` is the **LP-owned** reserve, while `protocol_fees_owed` is separate. Under normal operation without unsolicited transfers or issuer-side balance changes:

```text
vault token amount = actual_balance + protocol_fees_owed
```

Do not subtract protocol fees from `actual_balance` a second time. Collecting protocol fees transfers only the owed amount and clears that counter; it does not reduce actual or virtual balances. Direct transfers to vaults do not mint BPT or update these counters.

Pool asset mints can belong to classic SPL Token or Token-2022, independently per slot. The BPT mint also has its own token-program owner. Derive ATAs with the correct token program instead of assuming classic SPL Token. Creation-time extension admission and runtime transfer support are separate questions; the SDK conservatively rejects unsupported transfer paths. See [token policy](../safety/pool-controls.md#token-extension-policy).

## Authority boundaries

Pool admins, the range manager, the config's protocol authority, Treasury admin and supervisor are distinct roles. A pool creator is not automatically a Treasury admin, and Treasury PDA ownership does not imply a multisig. A wallet signs an outer Protocol Admin instruction; that program authorizes the signer and makes the Treasury PDA sign the inner CPI. A PDA cannot sign a top-level wallet transaction.

The supervisor in this branch can **freeze and unfreeze** pools and **disable and enable** token inputs. Program freeze (`freeze_pool_program`) means permanently removing upgrade authority; pool freeze (`freeze_pools`) only changes pool operating flags. Their effects are different. The complete permissions and limitations are in [pool controls](../safety/pool-controls.md).

## Reading and building

Use `CubicPoolClient.sync()` for a quote-ready snapshot: it combines pool state, mint metadata, BPT supply/program and chain time. `decodeContractAccount` exposes the exact stored ABI; check RPC account ownership as well as its discriminator. `get_pool_info` emits a convenient but incomplete `PoolInfo` event: it omits admin, range-manager, selloff and surge-policy fields. Read the account itself for those fields.

Use `buildContractInstruction(config, program, instruction, args, accounts, remainingAccounts)` for any of the 59 instructions. Its program selectors are `cubicPool`, `protocolAdmin`, and `singleTokenLiquidity`; argument and account keys use IDL snake_case. Integers wider than 32 bits are `BN`, pubkeys are `PublicKey`, and optional absent arguments are `null`. Higher-level clients add account derivation and quote guards, while the generic encoder only checks ABI shape and widths. It does not grant authority or bypass runtime constraints. See [SDK](../sdk/index.md).

## Migration compatibility

Current pool accounts remain **1,683 bytes**, config accounts **202 bytes**. The v4-to-v5 update reinterpreted per-token bytes and used reserved storage. `migrate_to_v5(reactivate_tokens)` backfills a zero pool policy bitmap from its config and, only when requested, sets every token input-active flag to true. It does not resize accounts or rewrite balances.

Use `reactivate_tokens=true` only for the verified one-time legacy transition. Repeating it later can reopen deliberately disabled tokens. Backfill is guarded by `pool.banned_extensions == 0`; this is a conditional write, not a version marker that proves a pool was migrated. The instruction does **not** accept the older 1,154-byte v3 format. Neither equal account length nor a zero reserved field alone establishes migration readiness; inspect the pre-upgrade state and preserve policy decisions.
