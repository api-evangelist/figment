---
name: Exit Ethereum validators and reconcile withdrawals with Figment
description: >-
  Request voluntary exits for Ethereum validators through Figment, broadcast the signed exit message,
  and reconcile the resulting withdrawals back to the withdrawal address.
api: openapi/figment-api-openapi-original.yml
generated: '2026-08-04'
method: generated
source: >-
  Grounded in the operationIds published in https://api.figment.io/openapi/figment-api.yaml and the
  exit/withdrawal guides at https://docs.figment.io/docs/exiting-eth-validator
operations:
  - exit-validators
  - broadcast-exit-message
  - get-validators
  - get-validator
  - eth-withdrawals
  - ethereum-activities
  - ethereum-withdrawal-transaction
---

# Exit Ethereum validators and reconcile withdrawals with Figment

An exit is irreversible and subject to the beacon chain exit queue. Read
`https://docs.figment.io/docs/queues-activation-and-deactivation` before running this at scale.

## Before you start

- `x-api-key` header, **Read/Write** key. Read-Only keys are explicitly rejected on
  `exit-validators`.
- Environment must match: `test` key → `hoodi`, `production` key → `mainnet`.

## Step 1 — Request the exit

`exit-validators` — `POST /ethereum/validators/exits`. The body is a `oneOf`, so pick one selector:

- **By public keys**: `{ "network": "...", "pubkeys": ["0x…", …] }` — every pubkey must be `0x`-prefixed.
- **By withdrawal address**: `{ "network": "...", "withdrawal_address": "0x…" }` — exits the
  validators under that address.

Do not send both selectors. Confirm the target set with `get-validators` first (filter on
`withdrawal_address` and `status`) so you exit exactly the intended validators.

## Step 2 — Sign and broadcast the exit message

An Ethereum voluntary exit is a **signed beacon-chain message**, not an execution-layer transaction.
Sign the exit message returned in step 1 with the validator key material held by your signer, then:

`broadcast-exit-message` — `POST /ethereum/broadcast_exit_message`, with all four required fields:
`network`, `epoch`, `validator_index`, `signature`.

Because this is an off-chain beacon message, the resulting `unstake` activity has `tx.hash` and
`tx.status` of `null` — that is expected and documented, not a failure.

## Step 3 — Watch the exit progress

`get-validator` — `GET /ethereum/validators/{pubkey}`, or `get-validators` filtered by `status`.
The status walk is `active_ongoing → active_exiting → exited_unslashed → withdrawal_possible →
withdrawal_done`. Add `include_fields=balances` to see `pending_withdrawals` accumulate.

`ethereum-activities` — `GET /ethereum/activities` — records the `unstake`; a single activity can be
fetched by UUID or transaction hash with `ethereum-activity-v2` (`GET /ethereum/activities/{id}`).

## Step 4 — Reconcile withdrawals

`eth-withdrawals` — `POST /ethereum/withdrawals`. Required: `start` and `end`. Optional: `network`,
`withdrawal_addresses`, `pubkeys`. This returns the sweeps that actually landed at the withdrawal
address, which is what you reconcile against — not the exit request.

For 0x02 compounding validators, a *partial* withdrawal does not require an exit at all: use
`ethereum-withdrawal-transaction` (`POST /ethereum/withdrawal`) to build a partial-withdrawal
transaction, sign it, and broadcast with `broadcast-staking-tx`. See the recipe
`https://docs.figment.io/recipes/0x02-partial-withdraw-ethereum-from-hot-wallet`.

## Conventions and failure handling

- Exits are **not** idempotency-protected — `X-Figment-Idempotency-Key` covers only the two
  provisioning operations. Never blind-retry an exit request; re-read validator status first.
- `401` = bad key, wrong environment, or Read-Only key. `422` = unsupported `network`.
  `404` on an activity lookup can also mean a feature-flagged endpoint hidden from your org.
- Errors are the proprietary `{error:{message, details[]}}` envelope, not RFC 9457.
- Rate limits: 200 req/s, 3500 req/min, no headers returned.
