---
name: Operate Figment Validator Vaults for omnibus ETH staking
description: >-
  Create and configure a Figment Validator Vault, manage its allowlist, take deposits and withdrawals
  on behalf of many end users from one wallet, and read per-address balances, rewards and exit
  requests.
api: openapi/figment-api-openapi-original.yml
generated: '2026-08-04'
method: generated
source: >-
  Grounded in the operationIds published in https://api.figment.io/openapi/figment-api.yaml and the
  omnibus guide at https://docs.figment.io/docs/0x02-eth-omnibus-staking
operations:
  - list-vaults
  - create-vault-transactions
  - configure-figment-vault
  - set-region
  - show-allowlist
  - update-allowlist-transactions
  - deposit-transactions
  - create-withdraw-transactions
  - create-claim-withdrawal-transactions
  - create-withdraw-commission-transactions
  - show-address-balance
  - show-address-actions
  - show-address-exit-requests
  - list-address-rewards
  - show-commission-balance
  - show-commission-actions
  - show-commission-exit_requests
  - assets-to-shares
  - shares-to-assets
---

# Operate Figment Validator Vaults for omnibus ETH staking

A vault is the omnibus pattern: **one wallet, many end users**. Your platform holds the wallet, the
vault contract tracks each depositor's share, and Figment runs the validators behind it. This is the
right surface when you are a custodian, exchange or wallet staking on behalf of customers rather than
staking your own treasury.

Read `https://docs.figment.io/docs/0x02-eth-omnibus-staking` for the architecture before you build.

## Before you start

- `x-api-key` header, **Read/Write** key for the transaction-building operations.
- Every write returns an **unsigned transaction** — you sign in your own custody and broadcast it
  yourself. The vault operations are execution-layer contract calls.
- Vault addresses are the primary key: nearly every path is
  `/vaults/{vault_address}/…`.

## Step 1 — Create and configure the vault

- `create-vault-transactions` — `POST /vaults/transactions/create_vault`. Returns the unsigned
  vault-deployment transaction.
- `configure-figment-vault` — `POST /vaults/{vault_address}/transactions/configure_figment_vault`.
  Points the vault at Figment as validator operator.
- `set-region` — `PATCH /vaults/{vault_address}`. Sets the infrastructure region
  (`ca-central-1`, `eu-west-1`) — relevant for jurisdictional requirements.
- `list-vaults` — `GET /vaults`, filterable by `network`. Each `vault` carries `address`,
  `fee_recipient_address`, `region`, `vault_admin_address`, `validators_manager_address`, `fee_bps`,
  `capacity_wei`, `own_mev_escrow` and `reward_splitter_contract_address`.

## Step 2 — Control who can deposit

- `show-allowlist` — `GET /vaults/{vault_address}/allowlist`. Entries carry `address` and
  `created_at`.
- `update-allowlist-transactions` — `POST /vaults/{vault_address}/transactions/update_allowlist`.
  Returns the unsigned transaction that mutates the on-chain allowlist.

The allowlist is enforced by the contract, not by the API — an unallowlisted deposit fails on-chain,
not with a Figment 4xx. Update the allowlist and confirm the transaction is mined *before* taking the
deposit.

## Step 3 — Deposit

`deposit-transactions` — `POST /vaults/{vault_address}/transactions/deposit`. All four fields are
required: `network`, `from_address`, `receiver_address`, `amount`.

`from_address` is your omnibus wallet; `receiver_address` is the end user whose position is being
credited. Getting these the wrong way round credits the wrong party, so validate against your own
ledger before signing.

Use `assets-to-shares` (`POST /vaults/{vault_address}/information/assets_to_shares`) and
`shares-to-assets` (`POST /vaults/{vault_address}/information/shares_to_assets`) to convert between
ETH amounts and vault shares — do not compute the conversion yourself, the exchange rate moves with
accrued rewards.

## Step 4 — Withdraw

Withdrawal is two-phase, because ETH must leave the beacon chain first:

1. `create-withdraw-transactions` — `POST /vaults/{vault_address}/transactions/withdraw`. Required:
   `network`, `from_address`, `receiver_address`, `assets`. This queues an exit request and returns a
   `position_ticket`.
2. `create-claim-withdrawal-transactions` —
   `POST /vaults/{vault_address}/transactions/claim_withdrawal`. Required: `network`,
   `position_ticket`, `timestamp`, `from_address`. Only claimable once the exit has settled.

Track the wait with `show-address-exit-requests`
(`GET /vaults/{vault_address}/address/{id}/exit_requests`). Do not poll the claim endpoint as a
substitute for reading exit-request state.

## Step 5 — Read per-address state

- `show-address-balance` — `GET /vaults/{vault_address}/address/{id}/balance`
- `show-address-actions` — `GET /vaults/{vault_address}/address/{id}/actions`
- `list-address-rewards` — `GET /vaults/{vault_address}/address/{id}/rewards`
- `show-address-exit-requests` — `GET /vaults/{vault_address}/address/{id}/exit_requests`

These are your per-end-user reporting surface for an omnibus book.

## Step 6 — Commission

The vault operator's own economics use the parallel commission endpoints:
`show-commission-balance`, `show-commission-actions`, `show-commission-exit_requests`
(`/vaults/{vault_address}/commission/{commission_address}/…`), and
`create-withdraw-commission-transactions`
(`POST /vaults/{vault_address}/transactions/withdraw_commission`).

## Conventions and failure handling

- Vault operations are **not** idempotency-protected. Since every write returns an unsigned
  transaction rather than executing one, a duplicate API call is cheap — but a duplicate *broadcast*
  is not. Deduplicate at the signing/broadcast layer.
- `404 Vault not found` is declared; a `404` can also mean the endpoint is feature-flagged off for
  your organization.
- Schema names in the contract are `stake_wise_*` (the underlying vault contracts) while the paths
  and docs say Vaults — same entity, different vocabulary.
- Rate limits: 200 req/s, 3500 req/min, unsignalled.
