# Recipes

Idiomatic patterns for common wallbit-cli workflows. Pick the closest one and adapt.

## Table of contents

- [1. Chained reads with interpolation](#1-chained-reads-with-interpolation)
- [2. Safe dev smoke test (continue on error)](#2-safe-dev-smoke-test-continue-on-error)
- [3. Pre-flight validate then run](#3-pre-flight-validate-then-run)
- [4. Multi-balance snapshot piped to jq](#4-multi-balance-snapshot-piped-to-jq)

---

## 1. Chained reads with interpolation

When a downstream step needs a value produced by an earlier read. Uses standalone refs so types pass through unchanged.

```yaml
version: 1
name: chained-reads
on_error: fail_fast

steps:
  - id: fx
    run: rates.get
    with:
      source: USD
      dest: EUR

  - id: fx_reverse
    run: rates.get
    with:
      source: ${steps.fx.data.Data.DestCurrency}
      dest: ${steps.fx.data.Data.SourceCurrency}

  - id: bank
    run: account_details.get
    with:
      country: US
      currency: ${steps.fx.data.Data.SourceCurrency}

  - id: tx
    run: transactions.list
    with:
      page: 1
      limit: 10
      currency: ${steps.fx.data.Data.SourceCurrency}
```

Use this when one piece of data (a currency code, a symbol, a flag) drives several follow-up calls.

## 2. Safe dev smoke test (continue on error)

Exercises every supported run against a dev account without aborting on transient failures. Mutations use the smallest possible amount; `card_uuid` and `robo_advisor_id` are placeholders the user must fill before running.

```yaml
version: 1
name: dev-smoke
on_error: continue   # one bad step does not stop the rest

steps:
  # --- reads ---
  - id: fx_usd_eur
    run: rates.get
    with: { source: USD, dest: EUR }

  - id: balance_checking
    run: balance.get_checking

  - id: balance_stocks
    run: balance.get_stocks

  - id: wallets_usdc_polygon
    run: wallets.get
    with: { currency: USDC, network: polygon }

  - id: asset_aapl
    run: assets.get
    with: { symbol: AAPL }

  - id: assets_page
    run: assets.list
    with: { page: 1, limit: 5 }

  - id: account_details_usd
    run: account_details.get
    with: { country: US, currency: USD }

  - id: transactions_recent
    run: transactions.list
    with: { page: 1, limit: 10, currency: USD }

  - id: cards_list
    run: cards.list

  # --- writes (smallest meaningful amounts) ---
  - id: trade_small_usd
    run: trades.create
    with:
      symbol: AAPL
      direction: BUY
      currency: USD
      order_type: MARKET
      amount: 1

  # Replace robo_advisor_id with your portfolio id.
  - id: robo_deposit_small
    run: roboadvisor.deposit
    with: { robo_advisor_id: 1, amount: 1, from: DEFAULT }

  - id: robo_withdraw_small
    run: roboadvisor.withdraw
    with: { robo_advisor_id: 1, amount: 1, to: DEFAULT }

  # Replace with a real UUID from cards_list output (refs cannot index arrays).
  - id: card_block
    run: cards.block
    with: { card_uuid: "REPLACE_WITH_CARD_UUID" }

  - id: card_unblock
    run: cards.unblock
    with: { card_uuid: "REPLACE_WITH_CARD_UUID" }

  # apikey.revoke deliberately omitted — destructive.
```

Run with `wallbit workflow run dev-smoke.yaml | jq` and inspect each `steps[].ok`.

## 3. Pre-flight validate then run

Validation catches every static error (unsupported `run`, missing `with`, duplicate `id`, wrong `version`, bad `on_error`) without touching the API. Always validate before running, especially in CI.

```bash
# Bash / Linux / macOS
wallbit workflow validate wf.yaml && wallbit workflow run wf.yaml
```

```powershell
# PowerShell — chain on success
if (wallbit workflow validate wf.yaml) { wallbit workflow run wf.yaml }
```

`validate` exits non-zero on failure; combine with `&&` (or `if` in PowerShell) so the run never starts with a broken spec.

## 4. Multi-balance snapshot piped to jq

A single workflow that gathers all the read data needed for a daily snapshot, then post-processes the JSON output without any further API calls.

```yaml
version: 1
name: snapshot
on_error: continue

steps:
  - id: checking
    run: balance.get_checking

  - id: stocks
    run: balance.get_stocks

  - id: fx_usd_eur
    run: rates.get
    with: { source: USD, dest: EUR }

  - id: tx_recent
    run: transactions.list
    with: { page: 1, limit: 25 }

  - id: cards
    run: cards.list
```

Pipe the JSON to `jq` (or `ConvertFrom-Json` in PowerShell) for shaping:

```bash
wallbit workflow run snapshot.yaml \
  | jq '{
      checking: (.steps[] | select(.id=="checking") | .data.Data),
      stocks:   (.steps[] | select(.id=="stocks")   | .data.Data),
      usd_eur:  (.steps[] | select(.id=="fx_usd_eur") | .data.Data.Rate),
      tx_count: (.steps[] | select(.id=="tx_recent") | .data.Data.Count),
      cards:    (.steps[] | select(.id=="cards")    | .data.Data | length)
    }'
```

```powershell
$snapshot = wallbit workflow run snapshot.yaml | ConvertFrom-Json
$snapshot.steps | ForEach-Object { "$($_.id): ok=$($_.ok) duration=$($_.duration_ms)ms" }
```

Because all five steps are reads with no cross-references, `on_error: continue` lets a single API hiccup degrade gracefully instead of dropping the rest of the snapshot.
