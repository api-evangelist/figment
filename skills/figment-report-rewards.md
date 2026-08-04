---
name: Report staking rewards and reconcile statements with Figment
description: >-
  Pull rewards, reward rates, withdrawals, portfolio and monthly statements across every protocol
  Figment supports, for accounting, client reporting and reconciliation.
api: openapi/figment-api-openapi-original.yml
generated: '2026-08-04'
method: generated
source: https://api.figment.io/openapi/figment-api.yaml
operations:
  - eth-rewards
  - get-ethereum-net-rewards
  - eth-rewards-rates
  - eth-withdrawals
  - sol-rewards
  - sol-rewards-rates
  - ada-rewards
  - get-cardano-rewards
  - atom-rewards
  - near-rewards
  - polkadot-rewards
  - matic-rewards
  - get-avalanche-rewards
  - get-injective-rewards
  - get-sui-rewards
  - get-opentrade-rewards
  - get-statements
  - portfolio
  - tracked-address-create
---

# Report staking rewards and reconcile statements with Figment

Rewards reporting is protocol-partitioned: there is no single cross-chain rewards endpoint. Pick the
operation for each protocol you hold, then roll up client-side — or use `portfolio` and
`get-statements` for the pre-aggregated view.

## Before you start

- `x-api-key` header. A **Read-Only** key is sufficient for everything in this skill — use one.
- Reporting reads what Figment can see for your organization. If a wallet is not tracked, its rewards
  will not appear.

## Step 0 — Make sure the addresses are tracked

`tracked-address-create` — `POST /addresses`. As of the 2025-11-14 changelog entry, `address_type` is
no longer required and the semantics are wallet-first:

- **Solana**: track a *wallet*; Figment auto-tracks every stake account for which that wallet is the
  **stake authority**.
- **Ethereum**: you can no longer track a single validator — track a *wallet*, which Figment
  interprets as the validators' **withdrawal address**.

A `409 Already tracked` is a success condition, not an error.

## Step 1 — Pull rewards per protocol

| Protocol | Operation | Method / path |
|---|---|---|
| Ethereum (gross) | `eth-rewards` | `POST /ethereum/rewards` |
| Ethereum (net of fees) | `get-ethereum-net-rewards` | `POST /ethereum/rewards_net` |
| Solana | `sol-rewards` | `POST /solana/rewards` |
| Cardano | `ada-rewards` / `get-cardano-rewards` | `POST /cardano/rewards`, `POST /cardano/rewards_all` |
| Cosmos | `atom-rewards` | `POST /cosmos/rewards` |
| NEAR | `near-rewards` | `POST /near/rewards` |
| Polkadot | `polkadot-rewards` | `POST /polkadot/rewards` |
| Polygon | `matic-rewards` | `POST /polygon/rewards` |
| Avalanche | `get-avalanche-rewards` | `POST /avalanche/rewards` |
| Injective | `get-injective-rewards` | `POST /injective/rewards` |
| Sui | `get-sui-rewards` | `POST /sui/rewards` |
| OpenTrade stablecoin vaults | `get-opentrade-rewards` | `POST /opentrade/rewards` |

`eth-rewards` requires `time_rollup`, `start` and `end`, and optionally takes `groups`, `network`,
`pubkeys`, `withdrawal_addresses`, `page` and `include_penalties`. Set `include_penalties` when you
need a true net figure — penalties are excluded by default.

`sol-rewards` takes an `anyOf` body: choose exactly one of by-stake-account, by-stake-authority,
by-withdraw-authority or by-groups, each with the common `start`/`end` window.

Because these are POST endpoints that paginate, pass pagination in the **body**:
`{"page": {"number": 2, "size": 100}}`.

## Step 2 — Reward rates, for forward-looking reporting

`eth-rewards-rates` (`GET /ethereum/rewards_rates`), `sol-rewards-rates`, `polkadot-rewards-rates`,
`get-cardano-rewards-rates`, `cardano-srr`, `get-sui-rewards-rates`, and `opentrade-apr`
(`GET /opentrade/{vault_address}/apr`) for stablecoin vault APR.

Rates are rate *observations*, not guarantees — do not present them as yield promises.

## Step 3 — Reconcile against what actually landed

Rewards accrued and rewards received are different numbers. For Ethereum, `eth-withdrawals`
(`POST /ethereum/withdrawals`, required `start`/`end`) returns the sweeps that actually reached the
withdrawal address. Reconcile accrual against settlement rather than reporting accrual alone.

For Cardano, `cardano-rewards-withdrawals` (`GET /cardano/rewards_withdrawals`) and
`cardano-transfers` (`GET /cardano/transfers`) play the same role.

## Step 4 — Portfolio and statements

- `portfolio` — `GET /portfolio`. No parameters. The cross-protocol position rollup for the calling
  organization.
- `get-statements` — `GET /statements`, filtered by `month` and paginated by `page`. Returns
  `monthly_statement` objects carrying `csv_data` — the artifact to hand to accounting.

## Conventions and failure handling

- Envelope `{data, meta}`; pagination in `meta.pagination` (`current_page`, `total_pages`,
  `total_item_count`), default page size 50, max 100.
- Rewards endpoints return the **thinnest** error envelope in the API — `{error:{message}}` with no
  `param`, no code and no request id. Log your own request context; you will not get correlation help
  from the response.
- `401` = bad key or an environment mismatch (a `test` key against mainnet data returns nothing
  useful). `422` = unsupported `network` for that protocol.
- Rate limits: 200 req/s, 3500 req/min. Large historical pulls should be chunked by date range and
  paced — no RateLimit headers are returned to steer you.
