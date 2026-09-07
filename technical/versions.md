# Versions and compatibility

This documentation was checked on **2026-09-07** against these source snapshots:

| Component | Source revision | Scope |
| --- | --- | --- |
| Contracts | [`audit-fixes-excluded-SF`, `96a2ee2`](https://github.com/coffer-so/contracts/tree/96a2ee20244ff95fb9f14357bb55b17e1eb0e2c0) | All three programs, instruction handlers, account layouts, events and math |
| SDK | [`main`, `27de819`, package version 0.11.1](https://github.com/coffer-so/sdk/tree/27de819c469056bfb7cd3ab3a4cfdbde741db2f8) | Clients, builders, all three IDLs, parsers, math and backend method declarations |
| Backend | [`v5.1`, `a886497`](https://github.com/coffer-so/backend-v2/tree/a8864979a70b73d266a5ee0b88e9145e88354974) | Local route/auth, router, analytics and XP implementation used to check the API pages |

A package version here identifies source, not an assertion about the current
npm dist-tag. Check the package actually installed by the application. A branch
name can move; commit IDs identify the implementation these pages describe.
Local backend source also does not identify which revision a live server runs.

## Contract and SDK surface

The SDK's three IDLs structurally match the checked contract build. The typed
instruction API covers 28 cubic-pool instructions, 29 protocol-admin instructions
and 2 single-token-liquidity instructions. It also decodes all 60 declared events.
The references document each argument, account, authority and emitted structure.

Encoding an instruction is different from satisfying its runtime constraints.
Generic construction does not generate PDA signatures, discover arbitrary
remaining accounts, create all required accounts, reserve a quote or guarantee
transaction size. Normal high-level builders apply additional integration
checks; the contract remains authoritative for execution.

The SDK validation at this revision included 502 passing tests, including ABI
encoding, account layouts, 179 native Rust math reference vectors, sequential
single-token quote examples and read-only mainnet checks. This is not proof that
every administrative operation has been executed against production, or a
replacement for a contract security audit.

## Storage and migration

The current pool account is 1,683 bytes, including its discriminator. V5 reuses
space reserved in the earlier layout of the same size. The SDK parses this
current schema; legacy v3 at 1,154 bytes is explicitly unsupported.

`migrate_to_v5(reactivate_tokens)` is size-preserving. It is not a converter from
v3 and does not add a reallocation path. Its token activation and bitmap
backfill effects are documented in the [instruction reference](instruction-reference.md).
Existing config values are not automatically replaced by fresh-config defaults.

## Boundaries to preserve in integrations

- Base fee and its protocol share round upward in the input token. Surge is a
  separate output-token fee with a four-segment calculation. Its entire amount
  belongs to the protocol.
- LP actual balances already exclude protocol fees. Quotes and position math
  must not subtract the fee bucket again.
- Quotes use state and time from a snapshot. Another transaction, window
  rotation, token change or administrative update can change the result before
  execution. Minimum output/BPT values and refreshed state remain necessary.
- STLD quotes assume zero existing helper balances unless the caller provides
  them. The actual helper reads its accounts and can include prior dust or
  donations. Its quote/normal-builder extension checks are intentionally not
  identical in every no-transfer edge case.
- Mint admission policy and runtime token-program transfer support are separate.
  This without-SF implementation does not provide transfer-fee accounting or
  arbitrary transfer-hook remaining accounts.
- The contract has a small-raw-amount proportional-deposit rounding limitation.
  The SDK rejects zero-rounded live transfer legs in its ordinary path, but that
  does not fix the contract. See [the LP limitation](../for-lps/liquidity.md#rounding-limit-in-this-contract-version).
- Some SDK backend methods target routes absent from the checked backend tree.
  See [route availability](../integration/api-reference.md); do not advertise a
  method as live solely because a TypeScript wrapper exists.

These pages describe executable checks rather than assuming all comments,
legacy documentation or front-end behavior still match the current code.
