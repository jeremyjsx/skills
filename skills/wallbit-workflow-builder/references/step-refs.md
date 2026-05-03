# Cross-step references

Workflow steps can read values from earlier steps using `${steps.<step_id>.<path>}` inside any `with` value. The runner resolves these before invoking the step handler.

## Syntax

```
${steps.<step_id>.<path>}
```

- `<step_id>` matches the regex `[a-zA-Z0-9_\-]+`.
- `<path>` is a dot-separated chain of field names.

## Resolution rules

1. **Only prior steps are visible.** Steps execute top-to-bottom; a ref to a later or same-step id fails with `reference step "<id>" not found`.
2. **Field names are Go struct names**, matched case-insensitively. They are NOT JSON keys.
   - JSON shows `"source_currency"` → reference path uses `SourceCurrency` (or `sourcecurrency`, `Sourcecurrency`, etc.).
   - The wrapping field is always `Data` in Go (top-level envelope on most SDK responses).
3. **The first segment is `data`** (resolves to `StepResult.Data` on the prior step). The runner also recognises `id`, `run`, `ok`, `error`, `duration_ms` here, but `data` is the only segment you'll need in practice.
4. **No array indexing.** There is no `[0]` syntax. If a path lands on a slice (`[]Wallet`, `[]Card`, `[]Transaction`, `[]CheckingBalance`, etc.), you cannot descend further.
5. **Pointers are auto-dereferenced.** Nullable pointer fields (`*string`, `*float64`, `*time.Time`) work transparently when non-nil; resolution fails if the pointer is nil.
6. **Failed lookups fail the step**, not just the reference. The error surfaces as `reference steps.<id>.<path> not found`. Under `on_error: fail_fast` this stops the workflow.

## Standalone vs embedded form

The runner detects two modes automatically based on whether the ref is the *entire* string or just a substring.

### Standalone (single ref, whole value)

When the entire `with` value is exactly one ref, the resolved value is passed through with its **native Go type** (string, int, float, bool, time, struct, slice, map, etc.).

```yaml
- id: tx_filtered
  run: transactions.list
  with:
    currency: ${steps.fx.data.Data.SourceCurrency}   # passed as string
    limit: ${steps.cfg.data.PageSize}                # passed as int
```

This is the form to use when the downstream handler expects a non-string type (int for `limit`/`page`, float for `amount`/`shares`, etc.).

### Embedded (interpolation into a string)

When the ref is mixed with other text or with other refs, every match is `fmt.Sprint`'d and concatenated. The result is always a string.

```yaml
- id: log
  run: assets.get
  with:
    symbol: ${steps.cfg.data.Prefix}-${steps.cfg.data.Suffix}   # always a string
```

If the downstream handler needs the value typed (e.g. `page` must be int), use the standalone form instead.

## Worked examples (reads)

```yaml
version: 1
name: chained-reads
steps:
  - id: fx
    run: rates.get
    with:
      source: USD
      dest: EUR

  # Reverse pair, fed from the previous response:
  - id: fx_reverse
    run: rates.get
    with:
      source: ${steps.fx.data.Data.DestCurrency}
      dest: ${steps.fx.data.Data.SourceCurrency}

  # Same currency drives an account_details lookup:
  - id: bank
    run: account_details.get
    with:
      country: US
      currency: ${steps.fx.data.Data.SourceCurrency}

  # And a filtered transaction list:
  - id: tx
    run: transactions.list
    with:
      page: 1
      limit: 10
      currency: ${steps.fx.data.Data.SourceCurrency}
```

## What you cannot do

These all fail validation or runtime resolution — do not generate them:

```yaml
# ❌ Array indexing (refs cannot select an element):
card_uuid: ${steps.cards_list.data.Data[0].UUID}

# ❌ JSON keys instead of Go struct names:
currency: ${steps.fx.data.data.source_currency}

# ❌ Forward reference (s2 hasn't run yet when s1 executes):
- id: s1
  run: assets.get
  with:
    symbol: ${steps.s2.data.Data.Symbol}
- id: s2
  run: assets.get
  with:
    symbol: AAPL

# ❌ Self reference:
- id: loop
  run: assets.get
  with:
    symbol: ${steps.loop.data.Data.Symbol}
```

For the values that **are** reachable per run, see [runs-reads.md](runs-reads.md) and [runs-writes.md](runs-writes.md).
