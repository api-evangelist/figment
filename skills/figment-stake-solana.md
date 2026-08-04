---
name: Stake, unstake and withdraw SOL with Figment
description: >-
  Run the full Solana staking lifecycle through Figment — build a delegation transaction, sign it in
  your own custody, broadcast it, then deactivate and withdraw, including stake-account merge and
  split housekeeping.
api: openapi/figment-api-openapi-original.yml
generated: '2026-08-04'
method: generated
source: https://api.figment.io/openapi/figment-api.yaml
operations:
  - solana-stake
  - solana-broadcast
  - solana-stakes
  - solana-undelegate
  - solana-withdraw
  - build-solana-merge-tx
  - build-solana-split-tx
  - solana-activities
  - sol-rewards
  - sol-rewards-rates
---

# Stake, unstake and withdraw SOL with Figment

Solana staking is epoch-bound: a delegation warms up over an epoch boundary and a deactivation cools
down over one. Every step below returns an **unsigned** transaction that you sign yourself.

## Before you start

- `x-api-key` header on every request. Environment must match the network: `test` key → `devnet`,
  `production` key → `mainnet`.
- Figment supports durable nonces throughout — pass `nonce_account` and `nonce_authority` when your
  signing flow (e.g. an offline or multi-approver custody workflow) needs a transaction that stays
  valid beyond the normal blockhash window.

## Step 1 — Build the delegation

`solana-stake` — `POST /solana/stake`. Required: `funding_account`, `vote_account`, `amount_sol`,
`network`. Optional: `nonce_account`, `nonce_authority`.

The response is a `solana_transaction`: `unsigned_transaction_serialized` plus
`unsigned_tx_serialized_hex`, `unsigned_tx_serialized_base64`, `signing_payload`,
`last_valid_block_height`, the created `stake_account`, and `is_durable_nonce`. Pick the encoding
your signer expects.

Record the returned `stake_account` — it is the handle for every later operation.

## Step 2 — Sign, then broadcast

Sign with your wallet or custody provider (`@figmentio/slate` decodes and signs payloads; Fireblocks
raw-signing is covered at `https://docs.figment.io/recipes/stake-sol-from-fireblocks`).

`solana-broadcast` — `POST /solana/broadcast`. Required: `transaction_payload`, `network`.

If you are not using a durable nonce, respect `last_valid_block_height` — a transaction broadcast
after that height fails and must be rebuilt from step 1, not retried.

## Step 3 — Read your positions

`solana-stakes` — `GET /solana/stakes`. Filter by `network`, `groups`, `stake_authority`,
`withdraw_authority` (the authority filters were added 2025-09-22). Each
`tracked_solana_stake_account` carries `stake_account`, `vote_account`, `status`, `balance`,
`active_balance`, `inactive_balance` and their USD equivalents.

`solana-activities` — `GET /solana/activities` — audit trail. Solana activity `type` is one of
`delegation`, `undelegation`, `withdrawal`; all Solana activities are on-chain, so `tx.status` is
always meaningful (`in_progress`, `confirmed`, `failed`, `expired`). Fetch one with
`get-solana-activity` (`GET /solana/activities/{id}`) by UUID or transaction hash.

## Step 4 — Unstake

`solana-undelegate` — `POST /solana/undelegate`. Required: `stake_account`, `network`. Sign and
broadcast with `solana-broadcast`. The stake account deactivates at the next epoch boundary; the SOL
is not spendable until cooldown completes.

## Step 5 — Withdraw

`solana-withdraw` — `POST /solana/withdraw`. Required: `stake_account`, `recipient_account`,
`amount_sol`, `network`. Sign and broadcast. Only the inactive balance can be withdrawn — check
`inactive_balance` on `solana-stakes` before building the transaction, or the broadcast will fail
on-chain.

## Housekeeping — merge and split

- `build-solana-merge-tx` — `POST /solana/merge` — combine compatible stake accounts (same
  authorities, same delegation state) to reduce account sprawl.
- `build-solana-split-tx` — `POST /solana/split` — split a stake account when you need to withdraw or
  deactivate part of a position.

Both return unsigned transactions; sign and broadcast the same way.

## Rewards

- `sol-rewards` — `POST /solana/rewards`. The body is an `anyOf` over four selectors — by stake
  account, by stake authority, by withdraw authority, or by groups — each combined with the common
  `start`/`end` window. Send exactly one selector shape.
- `sol-rewards-rates` — `GET /solana/rewards_rates`.
- `solana-allocated-rewards` — `POST /solana/allocated_rewards`.
- To claim Jito MEV rewards, see `https://docs.figment.io/docs/claiming-jito-mev-rewards`.

## Conventions and failure handling

- No idempotency protection on any Solana operation — the two idempotency-protected endpoints are
  Ethereum validator provisioning only. Do not blind-retry; re-read `solana-stakes` first.
- Pagination: `page[number]` / `page[size]` (default 50, max 100).
- Errors: `{error:{message, details[]}}`; rewards endpoints return the thinner
  `{error:{message}}` shape with no machine-actionable field.
- Rate limits: 200 req/s, 3500 req/min, unsignalled in headers.
