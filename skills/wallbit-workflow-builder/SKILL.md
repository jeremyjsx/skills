---
name: wallbit-workflow-builder
description: Build, edit, and validate YAML workflow specs for the wallbit-cli `workflow run` command. Use when the user mentions wallbit workflow, workflow.yaml for wallbit, `wallbit workflow run`, `wallbit workflow validate`, wants to chain Wallbit API calls (rates, balance, wallets, assets, account_details, transactions, cards, trades, roboadvisor, apikey) declaratively, or pastes a YAML containing `version: 1` together with `steps:` referencing wallbit run ids like `rates.get`, `balance.get_checking`, `trades.create`, etc.
---

# wallbit-cli workflow builder

Wallbit-cli runs declarative YAML workflows that call the Wallbit API through a fixed registry of 15 run ids. This skill produces specs that pass `wallbit workflow validate` and execute as expected with `wallbit workflow run`.

## Quick start

Minimum valid spec (saved as `wf.yaml`):

```yaml
version: 1
name: my-workflow
steps:
  - id: bal
    run: balance.get_checking
```

Validate, then run:

```bash
wallbit workflow validate wf.yaml
wallbit workflow run wf.yaml
```

## Top-level fields

| Field      | Required | Notes                                                        |
| ---------- | -------- | ------------------------------------------------------------ |
| `version`  | yes      | Must be `1`. Any other value is rejected.                    |
| `name`     | yes      | Free-form string echoed back in the run output.              |
| `on_error` | no       | `fail_fast` (default) or `continue`.                         |
| `steps`    | yes      | Non-empty list. Each step must have unique `id` and a `run`. |

`on_error: fail_fast` stops at the first failing step. `on_error: continue` runs every step regardless; the final result is `ok: false` if any step failed.

## Step fields

| Field  | Required | Notes                                                                                           |
| ------ | -------- | ----------------------------------------------------------------------------------------------- |
| `id`   | yes      | Must be unique. No format enforced; snake_case is the convention used in existing examples.     |
| `run`  | yes      | Must be a registered run id (see catalog below).                                                |
| `with` | no       | Map of inputs. Required keys depend on `run`. **Omit `with:` entirely** when the run has no inputs (e.g. `balance.get_checking`, `balance.get_stocks`, `cards.list`). |

## Run catalog

Use this only as an **index**. For required/optional `with` keys per run, open the matching reference file.

The **Shape** column tells you what the run output looks like and therefore which `${steps.<id>.…}` paths are reachable:

- `obj` — `data.Data` is a single struct. Fields are reachable directly under `data.Data.<Field>`.
- `slice` — `data.Data` is `[]Something`. **Not indexable from refs** (no `[0]`).
- `paged-flat` — pagination metadata sits **alongside** the slice, one level deep: `data.Pages`, `data.CurrentPage`, `data.Count`, and `data.Data` (the slice). Currently only `assets.list`.
- `paged-nested` — pagination metadata sits **inside** a `Data` wrapper, two levels deep: `data.Data.Pages`, `data.Data.CurrentPage`, `data.Data.Count`, and `data.Data.Data` (the slice). Currently only `transactions.list`.

In every case the slice itself cannot be indexed from a ref. Always confirm exact paths against [references/runs-reads.md](references/runs-reads.md) for the run you are using.

**Reads** — see [references/runs-reads.md](references/runs-reads.md) for inputs and field paths:

| Run                    | Shape         | Summary                              |
| ---------------------- | ------------- | ------------------------------------ |
| `rates.get`            | obj           | FX rate between two currencies.      |
| `balance.get_checking` | slice         | Checking balances per currency.      |
| `balance.get_stocks`   | slice         | Stock positions.                     |
| `wallets.get`          | slice         | Crypto wallet addresses.             |
| `assets.list`          | paged-flat    | Paginated asset catalog.             |
| `assets.get`           | obj           | Single asset details.                |
| `account_details.get`  | obj           | Fiat banking details.                |
| `transactions.list`    | paged-nested  | Paginated transaction history.       |
| `cards.list`           | slice         | Issued cards.                        |

**Writes / mutations** — see [references/runs-writes.md](references/runs-writes.md):

| Run                                              | Shape | Summary                                                  |
| ------------------------------------------------ | ----- | -------------------------------------------------------- |
| `cards.block`, `cards.unblock`                   | obj   | Toggle card status.                                      |
| `trades.create`                                  | obj   | Submit a buy/sell order.                                 |
| `roboadvisor.deposit`, `roboadvisor.withdraw`    | obj   | Move funds in/out of a robo portfolio.                   |
| `apikey.revoke`                                  | obj   | Destructive; invalidates the current API key.            |

## Cross-step references

Inside any `with` value, use `${steps.<step_id>.<path>}` to read from a prior step's result. Example:

```yaml
- id: fx
  run: rates.get
  with:
    source: USD
    dest: EUR

- id: account
  run: account_details.get
  with:
    currency: ${steps.fx.data.Data.SourceCurrency}
```

Two non-obvious rules trip everyone the first time:

1. The path walks **Go struct field names** (case-insensitive), not JSON keys, so it is `data.Data.SourceCurrency`, not `data.data.source_currency`.
2. **Array indexing is not supported** (no `data[0]`).

For the full ref grammar, descent rules, standalone vs embedded modes, and per-run available paths, read [references/step-refs.md](references/step-refs.md).

## Validate before running

Always run validation first — it catches every static error (unsupported `run`, missing required `with` keys, duplicate ids, bad `on_error`, wrong `version`) without touching the API:

```bash
wallbit workflow validate path/to/wf.yaml
```

The validator exits non-zero on failure and prints the offending step index and reason.

**Validate does NOT resolve `${steps.<id>.<path>}` references.** A wrong path (typo, missing field, indexing into a slice) passes `validate` and only surfaces at `run` time as `reference steps.<id>.<path> not found`, failing the step. To catch ref bugs before production, do a smoke `run` against a dev account.

## Output shape

`validate` and `run` emit **different** JSON shapes. Both go to stdout pretty-printed.

`wallbit workflow validate wf.yaml` (small summary, no API calls):

```json
{ "name": "my-workflow", "ok": true, "steps": 2, "version": 1 }
```

`wallbit workflow run wf.yaml` (full execution result):

```json
{
  "workflow": "my-workflow",
  "ok": true,
  "started_at": "...",
  "finished_at": "...",
  "failed_step_id": "",
  "steps": [
    {
      "id": "bal",
      "run": "balance.get_checking",
      "ok": true,
      "data": { "...": "raw service payload" },
      "duration_ms": 123
    }
  ]
}
```

Note: the `name` field becomes `workflow` in the run output. On failure: `ok: false`, `failed_step_id` is set, and the failing step has `error.message` instead of `data`. Pipe to `jq` for ad-hoc inspection.

## Critical pitfalls

1. **`trades.create`** requires **exactly one** of `amount` (notional in `currency`) or `shares` (quantity). Providing both, or neither, fails validation.
2. **`roboadvisor.deposit` / `roboadvisor.withdraw`**: `from` / `to` must be the literal string `DEFAULT` or `INVESTMENT`, and `amount` must be `> 0`.
3. **`cards.block` / `cards.unblock`**: `card_uuid` must be a real UUID string. You **cannot** pull it from `${steps.cards_list.data.data[0].uuid}` — array indexing is unsupported. Either hardcode the UUID or split the workflow.
4. **`apikey.revoke`** is destructive — it invalidates the API key currently in use. Comment it out in any shareable example.

## Common patterns and recipes

For idiomatic patterns (chained reads, safe dev smoke tests, multi-balance snapshots, validate-then-run pre-flight), see [references/recipes.md](references/recipes.md).

## Authoring checklist

Before handing the file back to the user:

- [ ] `version: 1` set, `name` set.
- [ ] Every step `id` is unique.
- [ ] Every `run` is in the catalog above.
- [ ] All required `with` keys per run are present (check the reference file).
- [ ] `${steps.…}` paths reference only **prior** steps, use Go field names, no `[index]`.
- [ ] Mutations use small/safe values or are commented out by default.
- [ ] `wallbit workflow validate` passes locally if the env is set up.
