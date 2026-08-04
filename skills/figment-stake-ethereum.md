---
name: Provision and fund Ethereum validators with Figment
description: >-
  Provision Ethereum validators through Figment, sign the returned deposit transaction in your own
  custody, broadcast it, and confirm activation — including the idempotency discipline that stops a
  retry from provisioning duplicate validators.
api: openapi/figment-api-openapi-original.yml
generated: '2026-08-04'
method: generated
source: >-
  Grounded in the operationIds published in https://api.figment.io/openapi/figment-api.yaml and the
  semantics documented at https://docs.figment.io/reference/idempotency-requests
operations:
  - create-validators
  - create-pectra-validators
  - broadcast-staking-tx
  - get-validators
  - get-validator
  - ethereum-validators-summary
  - ethereum-activities
---

# Provision and fund Ethereum validators with Figment

Figment never holds your keys. Every write in this flow returns an **unsigned transaction**; you sign
it in your own custody (hot wallet, Fireblocks, Anchorage, BitGo, Taurus, Zodia) and post the signed
payload back to Figment to broadcast.

## Before you start

- Base URL: `https://api.figment.io`. HTTPS only — plain HTTP is 301-redirected.
- Every request needs the header `x-api-key: <your key>`.
- The key must be **Read/Write**. A Read-Only key is rejected on `create-validators` and
  `exit-validators`.
- The key's environment must match the network: a `test` key works only on `hoodi`, a `production`
  key only on `mainnet`.

## Step 1 — Decide 0x01 or 0x02

Two provisioning operations exist and they are not interchangeable:

- `create-validators` — `POST /ethereum/validators`. Standard 0x01 withdrawal credentials, fixed
  32 ETH per validator. Body takes `network`, `validators_count`, `withdrawal_address`,
  `funding_address`.
- `create-pectra-validators` — `POST /ethereum/validators/0x02`. Post-Pectra compounding validators.
  Body requires `network`, `withdrawal_address` and `amounts` (an array — validator sizes rather than
  a count), and optionally `funding_address`, `fee_recipient_address`, `region`.

Choose 0x02 if you want compounding balances and consolidation. See
`https://docs.figment.io/docs/0x02-eth-validator-balance-management`.

## Step 2 — Send the provisioning request WITH an idempotency key

Both provisioning operations are the only two idempotency-protected endpoints in the Figment API,
and that is deliberate: a retry without a key can provision extra validators, which is expensive and
hard to reverse.

- Generate a UUID v4 **once per logical provisioning operation** and store it client-side.
- Send it as `X-Figment-Idempotency-Key`.
- Retry with the **same key and the same body**. Same key + a *different* body returns **409**, and
  so does a retry while the first request is still in flight.
- Only 2xx responses are cached. If the call fails 4xx/5xx the key is released and the same key may
  be reused.

The response is `201 Created` with `data[]` of provisioned validators (`network`, `pubkey`,
`status`, `withdrawal_address`) and a `meta` block carrying `staking_request` and
`staking_transaction` — the unsigned deposit transaction.

Note: the `409` is documented but is **not** declared in the published OpenAPI, so handle it
explicitly rather than relying on generated client code.

## Step 3 — Sign in your own custody

Take `meta.staking_transaction` and sign it with your signer. Figment publishes signing guides at
`https://docs.figment.io/docs/signing-transactions` and, for Fireblocks,
`https://docs.figment.io/docs/signing-transactions-with-the-fireblocks-api`.

Verify before signing: use the Transaction Decoder
(`https://docs.figment.io/docs/transaction-decoder`) or `@figmentio/slate` to decode the payload and
confirm the deposit contract, amounts and withdrawal credentials are what you expect. This is a
one-way, irreversible operation — decode first.

## Step 4 — Broadcast

`broadcast-staking-tx` — `POST /ethereum/broadcast` with `network` and `signed_transaction`.
Returns the transaction hash and the resulting activity.

Broadcast is **not** idempotency-protected. Replay protection here is the chain's own nonce
semantics, so do not blind-retry a broadcast; re-read state first (Step 5).

## Step 5 — Confirm and monitor

- `get-validators` — `GET /ethereum/validators`. Filter by `network`, `withdrawal_address`,
  `status`, `credentials_prefix` (`0x01` or `0x02`), `staking_request_id`, `region`,
  `net_fee_payout_address`, `groups`. Add `include_fields=balances` to get
  `balances.actual`, `pending_deposit`, `pending_topups`, `pending_withdrawals` — these are opt-in
  and omitted by default.
- `get-validator` — `GET /ethereum/validators/{pubkey}` for one validator.
- `ethereum-validators-summary` — `GET /ethereum/validators_summary` for the aggregate.
- `ethereum-activities` — `GET /ethereum/activities` for the audit trail. Activity `type` is one of
  `stake`, `unstake`, `delegation`, `undelegation`, `consolidation`, `upgrade`; `tx.status` is
  `in_progress`, `confirmed`, `failed` or `expired`.

Validator status walks `provisioned → funding_requested → deposited → pending_queued →
active_ongoing`. Entry queue timing is a network property, not a Figment one — see
`https://docs.figment.io/docs/queues-activation-and-deactivation`.

## Conventions and failure handling

- **Pagination**: `page[number]` / `page[size]` (default 50, max 100). Response carries
  `meta.pagination.{current_page,total_pages,total_item_count}`.
- **Envelope**: `{data, meta}`.
- **Errors**: proprietary `{error:{message, details:[{param, message, code, context}]}}` — not
  RFC 9457. `401` = bad key, wrong environment, or a Read-Only key on a write op. `422` is most often
  an unsupported `network` value. See `errors/figment-problem-types.yml`.
- **Rate limits**: 200 req/s, 3500 req/min. No RateLimit-* headers are returned, so implement your
  own budget tracking and exponential backoff on `429`.
- **Correlation**: no request-id response header exists on this surface. Log your idempotency key and
  the `staking_request_id` as your own correlation handles.
