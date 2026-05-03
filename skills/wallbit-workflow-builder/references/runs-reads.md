# Read runs

Reference for the 9 non-mutating runs. Each entry lists `with` inputs and the **field paths** (Go struct names, case-insensitive) reachable from `${steps.<id>.data...}` for downstream interpolation.

> Reminder: all paths use Go struct field names (e.g. `Data`, `SourceCurrency`), NOT JSON keys. Array indexing is unsupported. See [step-refs.md](step-refs.md).

## Table of contents

- [`rates.get`](#ratesget)
- [`balance.get_checking`](#balanceget_checking)
- [`balance.get_stocks`](#balanceget_stocks)
- [`wallets.get`](#walletsget)
- [`assets.list`](#assetslist)
- [`assets.get`](#assetsget)
- [`account_details.get`](#account_detailsget)
- [`transactions.list`](#transactionslist)
- [`cards.list`](#cardslist)

---

## `rates.get`

FX rate between two currencies.

| Key      | Required | Type   | Notes                              |
| -------- | -------- | ------ | ---------------------------------- |
| `source` | yes      | string | ISO currency code, uppercased.     |
| `dest`   | yes      | string | ISO currency code, uppercased.     |

```yaml
- id: fx
  run: rates.get
  with:
    source: USD
    dest: EUR
```

Reachable downstream paths (via `${steps.fx.data...}`):

- `data.Data.SourceCurrency` (string)
- `data.Data.DestCurrency` (string)
- `data.Data.Pair` (string, e.g. `USD/EUR`)
- `data.Data.Rate` (float)
- `data.Data.UpdatedAt` (timestamp pointer; `null` for identity pairs like USD→USD)

## `balance.get_checking`

Checking balances per currency. No inputs.

```yaml
- id: checking
  run: balance.get_checking
```

Response is a **slice** (`Data: []CheckingBalance`). You **cannot** index it from another step. To use specific balances, parse the JSON output of the run and feed values manually, or restructure into multiple workflows.

Element shape (informational):

- `Currency` (string)
- `Balance` (float)

## `balance.get_stocks`

Stock positions. No inputs.

```yaml
- id: stocks
  run: balance.get_stocks
```

`Data: []StockPosition` — same array-indexing limitation as `balance.get_checking`.

Element shape:

- `Symbol` (string)
- `Shares` (float)

## `wallets.get`

Crypto wallet addresses. Both filters optional.

| Key        | Required | Type   | Notes                                                |
| ---------- | -------- | ------ | ---------------------------------------------------- |
| `currency` | no       | string | Uppercased (e.g. `USDC`).                            |
| `network`  | no       | string | Lowercased (e.g. `polygon`, `ethereum`, `solana`).   |

```yaml
- id: wallets
  run: wallets.get
  with:
    currency: USDC
    network: polygon
```

`Data: []Wallet` (no array indexing across steps). Element shape: `Address`, `Network`, `CurrencyCode`.

## `assets.list`

Paginated asset catalog. All inputs optional.

| Key        | Required | Type   | Notes                                                  |
| ---------- | -------- | ------ | ------------------------------------------------------ |
| `category` | no       | string | Uppercased category (e.g. `TECHNOLOGY`, `ETF`).        |
| `search`   | no       | string | Free-text search.                                      |
| `page`     | no       | int    | 1-based.                                               |
| `limit`    | no       | int    | Page size.                                             |

```yaml
- id: techs
  run: assets.list
  with:
    category: TECHNOLOGY
    page: 1
    limit: 5
```

Reachable paths from `${steps.techs.data...}`:

- `data.Pages` (int)
- `data.CurrentPage` (int)
- `data.Count` (int)
- `data.Data` is `[]Asset` — not indexable from refs.

> Known dev-env quirk: the API may return `dividend.yield` as a string while the SDK expects a number, causing `assets.list` to fail mid-workflow. Use `on_error: continue` if this is acceptable.

## `assets.get`

Single asset details.

| Key      | Required | Type   | Notes                                  |
| -------- | -------- | ------ | -------------------------------------- |
| `symbol` | yes      | string | Asset ticker, auto-uppercased.         |

```yaml
- id: aapl
  run: assets.get
  with:
    symbol: AAPL
```

Reachable paths from `${steps.aapl.data...}`:

- `data.Data.Symbol`, `data.Data.Name`, `data.Data.Price` (float)
- `data.Data.AssetType`, `data.Data.Exchange`, `data.Data.Sector`, `data.Data.Country`, `data.Data.MarketCapM`, `data.Data.LogoURL`
- `data.Data.Dividend.Amount` / `.Yield` / `.ExDate` / `.PaymentDate` (all may be null)

## `account_details.get`

Fiat banking details for the account.

| Key        | Required | Type   | Notes                                  |
| ---------- | -------- | ------ | -------------------------------------- |
| `country`  | no       | string | Uppercased (`US`, `EU`).               |
| `currency` | no       | string | Uppercased (`USD`, `EUR`).             |

```yaml
- id: bank
  run: account_details.get
  with:
    country: US
    currency: USD
```

Reachable paths from `${steps.bank.data...}`:

- `data.Data.BankName`, `data.Data.Currency`, `data.Data.AccountType`, `data.Data.HolderName`
- `data.Data.AccountNumber`, `data.Data.RoutingNumber`, `data.Data.IBAN`, `data.Data.BIC`, `data.Data.SWIFTCode` (nullable)
- `data.Data.Address.City`, `data.Data.Address.PostalCode`, `data.Data.Address.Country`, etc.

## `transactions.list`

Paginated transaction history. All inputs optional.

| Key        | Required | Type   | Notes                                                  |
| ---------- | -------- | ------ | ------------------------------------------------------ |
| `page`     | no       | int    | 1-based.                                               |
| `limit`    | no       | int    | Page size.                                             |
| `status`   | no       | string | E.g. `COMPLETED`, `PENDING`.                           |
| `type`     | no       | string | E.g. `DEPOSIT`, `WITHDRAWAL`, `TRADE`.                 |
| `currency` | no       | string | Filter by currency code.                               |

```yaml
- id: tx
  run: transactions.list
  with:
    page: 1
    limit: 10
    status: COMPLETED
    currency: USD
```

Reachable paths from `${steps.tx.data...}`:

- `data.Data.Pages`, `data.Data.CurrentPage`, `data.Data.Count` (int)
- `data.Data.Data` is `[]Transaction` — not indexable.

Note the doubled `Data.Data`: the outer is the SDK envelope, the inner is the pagination wrapper.

## `cards.list`

Issued cards. No inputs.

```yaml
- id: cards
  run: cards.list
```

`Data: []Card` (no array indexing across steps). Element shape: `UUID`, `Status`, `CardType`, `CardNetwork`, `CardLast4`, `Expiration` (nullable).

To block/unblock a card from a workflow, copy the UUID from the JSON output and hardcode it into a follow-up workflow — refs cannot select array elements.
