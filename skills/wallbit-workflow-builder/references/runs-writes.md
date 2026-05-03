# Write / mutation runs

Reference for the 6 mutating runs. Each one moves money, changes account state, or revokes credentials.

> **Safety rules — apply by default**
>
> 1. In any spec meant to be shared, committed, or reused as a template, **comment out** mutation steps with a placeholder for the dynamic value (uuid, robo_advisor_id).
> 2. Use the smallest meaningful amount on dev (e.g. `$1` notional, `1` share).
> 3. Never put `apikey.revoke` in a smoke test. Run it only intentionally, on its own.
> 4. Card UUIDs and robo_advisor_ids cannot be obtained from a prior step (no array indexing in refs). Source them out-of-band.

## Table of contents

- [`cards.block`](#cardsblock)
- [`cards.unblock`](#cardsunblock)
- [`trades.create`](#tradescreate)
- [`roboadvisor.deposit`](#roboadvisordeposit)
- [`roboadvisor.withdraw`](#roboadvisorwithdraw)
- [`apikey.revoke`](#apikeyrevoke)

---

## `cards.block`

Suspends a card.

| Key         | Required | Type   | Notes                                                |
| ----------- | -------- | ------ | ---------------------------------------------------- |
| `card_uuid` | yes      | string | Real UUID from `cards.list` JSON output.             |

```yaml
# - id: card_block
#   run: cards.block
#   with:
#     card_uuid: "REPLACE_WITH_CARD_UUID"
```

Reachable downstream paths from `${steps.<id>.data...}`:

- `data.Data.UUID` (string)
- `data.Data.Status` (string, `SUSPENDED` after success)

## `cards.unblock`

Reactivates a card. Same `with` shape and validation as `cards.block`.

```yaml
# - id: card_unblock
#   run: cards.unblock
#   with:
#     card_uuid: "REPLACE_WITH_CARD_UUID"
```

Response: `data.Data.Status == "ACTIVE"` on success.

## `trades.create`

Submits a buy/sell order.

| Key             | Required          | Type   | Notes                                                                          |
| --------------- | ----------------- | ------ | ------------------------------------------------------------------------------ |
| `symbol`        | yes               | string | Asset ticker, auto-uppercased (e.g. `AAPL`).                                   |
| `direction`     | yes               | string | `BUY` or `SELL`, auto-uppercased.                                              |
| `currency`      | yes               | string | Settlement currency, e.g. `USD`.                                               |
| `order_type`    | yes               | string | `MARKET`, `LIMIT`, `STOP`, etc., auto-uppercased.                              |
| `amount`        | XOR with `shares` | float  | Notional amount in `currency`. Use this OR `shares`, never both, never neither. |
| `shares`        | XOR with `amount` | float  | Number of shares.                                                              |
| `stop_price`    | no                | float  | Required for stop orders.                                                      |
| `limit_price`   | no                | float  | Required for limit orders.                                                     |
| `time_in_force` | no                | string | E.g. `DAY`, `GTC`, auto-uppercased.                                            |

**Hard rule:** validation rejects the step if both `amount` and `shares` are set, or if neither is.

```yaml
# Notional $1 market buy — smallest safe size for dev accounts.
- id: trade_small
  run: trades.create
  with:
    symbol: AAPL
    direction: BUY
    currency: USD
    order_type: MARKET
    amount: 1
```

Reachable downstream paths from `${steps.trade_small.data...}`:

- `data.Data.Symbol`, `data.Data.Direction`, `data.Data.Status`
- `data.Data.Amount` (float), `data.Data.Shares` (float)
- `data.Data.OrderType`, `data.Data.LimitPrice`, `data.Data.StopPrice`, `data.Data.TimeInForce`
- `data.Data.CreatedAt`, `data.Data.UpdatedAt`

## `roboadvisor.deposit`

Moves funds from a settlement account into a robo portfolio.

| Key               | Required | Type   | Notes                                                          |
| ----------------- | -------- | ------ | -------------------------------------------------------------- |
| `robo_advisor_id` | yes      | int    | Portfolio id. Source from the SDK's `roboadvisor/balance` API. |
| `amount`          | yes      | float  | Must be `> 0`.                                                 |
| `from`            | yes      | string | Literal `DEFAULT` or `INVESTMENT` (auto-uppercased).           |

```yaml
- id: robo_deposit_small
  run: roboadvisor.deposit
  with:
    robo_advisor_id: 1
    amount: 1
    from: DEFAULT
```

Reachable downstream paths:

- `data.Data.UUID` (string)
- `data.Data.Type` (string)
- `data.Data.Amount` (float)
- `data.Data.Status` (string)
- `data.Data.CreatedAt` (timestamp)

## `roboadvisor.withdraw`

Inverse of `roboadvisor.deposit`.

| Key               | Required | Type   | Notes                                                |
| ----------------- | -------- | ------ | ---------------------------------------------------- |
| `robo_advisor_id` | yes      | int    | Same id used for the deposit.                        |
| `amount`          | yes      | float  | Must be `> 0`.                                       |
| `to`              | yes      | string | Literal `DEFAULT` or `INVESTMENT` (auto-uppercased). |

```yaml
- id: robo_withdraw_small
  run: roboadvisor.withdraw
  with:
    robo_advisor_id: 1
    amount: 1
    to: DEFAULT
```

Same response shape as `roboadvisor.deposit`.

## `apikey.revoke`

Destructive. Invalidates the API key currently authenticating the CLI. After this step succeeds, every subsequent step in the workflow (and every later CLI call) will fail with an auth error until a new key is provisioned and configured.

No `with` inputs.

```yaml
# DANGER: leaves the CLI without credentials. Comment out by default.
# - id: revoke
#   run: apikey.revoke
```

Response: `data.Message` (string).

**Always** put `apikey.revoke` last in any workflow that includes it, and prefer running it as a single-step workflow on its own.
