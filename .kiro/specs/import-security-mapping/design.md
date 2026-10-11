# Design: Import Security Mapping

## Overview

Add an `Import::SecurityMapping` step to the web `TradeImport` flow that collects distinct
`ticker` labels from the CSV, auto-matches any that already resolve to a `Security` in the
local DB, and surfaces the rest for the user to map via the existing provider-backed
security-search combobox. `TradeImport#import!` then uses each row's resolved mapping
instead of resolving the raw `ticker` directly.

The design mirrors the established mapping pattern (`Import::AccountMapping`,
`Import::CategoryMapping`): a `Import::Mapping` subclass, a `securities` scope, inclusion in
`mapping_steps`, and rendering through the existing confirm/mappings partials. The only
genuinely new UI element is a searchable combobox in the mapping form (the other mappings
use plain selects over pre-populated lists).

## Architecture

### Data model

`Import::SecurityMapping < Import::Mapping` — no new table. It reuses the existing
`import_mappings` STI table and its columns:

- `key` — the raw CSV `ticker` label (e.g. `VANGUARD TOTAL BOND ETF MARKET` or `BND`).
- `mappable` (polymorphic) — the resolved `Security` once matched.
- `value` — the chosen combobox id `"SYMBOL|MIC|PROVIDER"` for user selections (parsed by
  `Security.parse_combobox_id`), unused for auto-matched rows.

Confirm the STI table already carries `key`, `value`, `mappable_type`, `mappable_id`,
`create_when_empty` (it does, per `AccountMapping`/`CategoryMapping` usage). No migration
expected; if a column is missing, add it in a dedicated task.

### Resolution strategy (two phases)

1. **Auto-match (local DB only, no network).** When mappings are built,
   `mappables_by_key` resolves each distinct ticker via `Security.find_by_ticker_and_exchange`
   (the same local lookup the resolver's `exact_match_from_db` uses). A hit assigns the
   `Security` as the mappable → auto-matched, no user action. This is cheap and avoids
   firing provider searches for every row on page load.
2. **Provider search (only for unresolved).** For labels with no local hit, the mapping is
   `requires_selection?` and renders the combobox, which calls the existing
   `SecuritiesController#index` → `Security.search_provider`. On selection, `create_mappable!`
   runs the full `Security::Resolver` with the chosen ticker/exchange/provider — this is the
   only place a network call happens, and only for labels the user actually maps.

### Mapping class contract

```ruby
class Import::SecurityMapping < Import::Mapping
  class << self
    def mappables_by_key(import)
      tickers = import.rows.map(&:ticker).compact.uniq
      tickers.index_with do |ticker|
        Security.find_by_ticker_and_exchange(ticker: ticker.strip.upcase, exchange_operating_mic: nil)
      end
    end
  end

  def selectable_values
    # Combobox is provider-backed/searchable, not a static list; the view renders a
    # security combobox rather than iterating selectable_values. Returned list is the
    # already-selected option (if any) so the control shows current state.
  end

  def requires_selection? = true

  def values_count
    import.rows.where(ticker: key).count
  end

  def mappable_class = Security

  def create_mappable!
    return unless value.present? # user picked a combobox option
    parsed = Security.parse_combobox_id(value)
    security = Security::Resolver.new(
      parsed[:ticker],
      exchange_operating_mic: parsed[:exchange_operating_mic],
      price_provider: parsed[:price_provider]
    ).resolve
    update!(mappable: security)
  end
end
```

Note: unlike `AccountMapping`/`CategoryMapping`, the mappable is a globally-scoped
`Security` (not family-scoped), so there is no "create new" (`CREATE_NEW_KEY`) path — the
resolver itself creates offline securities when needed.

### `TradeImport` wiring

```ruby
def mapping_steps
  base = []
  base << Import::AccountMapping if account.nil?
  base << Import::SecurityMapping
  base
end
```

In `import!`, resolve the security per row through the mapping, falling back to the current
direct resolver so any ticker already auto-matched (or an edge case with no mapping row)
still works:

```ruby
security = mappings.securities.mappable_for(row.ticker) ||
           find_or_create_security(ticker: row.ticker, exchange_operating_mic: row.exchange_operating_mic)
```

Because auto-matched mappings already hold the DB security and user-mapped ones hold the
resolved security, `mappable_for(row.ticker)` returns the right record in both cases.

### Scope

Add to `Import::Mapping`:

```ruby
scope :securities, -> { where(type: "Import::SecurityMapping") }
```

### Controller

`Import::MappingsController#update` already permits `:value`, `:mappable_id`,
`:mappable_type`, `:type`, `:key`. For a security mapping the client submits `value` (the
combobox id). The controller's current flow sets `mappable` via `mappable_class.find_by(id:)`
— that path doesn't fit a provider-search selection (no pre-existing id). Design decision:
the security mapping persists `value` and resolves `mappable` in `create_mappable!` at
`import!` time, OR resolve immediately on update. **Chosen:** resolve on update so the
confirm screen reflects a matched state right away. This means `MappingsController#update`
must call `create_mappable!` (or an equivalent) for security mappings after saving `value`.
Keep this change narrow and type-guarded so other mapping types are unaffected.

### View

- `app/views/import/mappings/_form.html.erb` gains a branch for `Import::SecurityMapping`
  that renders the security-search combobox (same Stimulus combobox + `securities_path`
  source the manual trade form uses) instead of a plain `select`. Auto-matched mappings
  render the matched security as the current value; unresolved ones render empty with the
  searchable control.
- Reuse DS primitives and the `icon` helper; no raw Tailwind palette or hex. Add i18n keys
  for the new titles/descriptions (`import_security_mapping_title`/`_description`) and the
  confirm `_mappings` labels.

## Correctness Properties

### Property 1: Clean CSVs require no interaction
For a `TradeImport` whose tickers all resolve locally, every security mapping is
auto-matched, so the step is non-blocking and published trades match current behavior.

### Property 2: Descriptive labels become real securities
A row labeled with a descriptive name, once mapped by the user, produces a `Trade`
referencing the user-selected `Security` (not an offline security named after the label).

### Property 3: Per-row resolution is order-independent
`import!` resolves each row through `mappings.securities.mappable_for(row.ticker)`; the
result is identical regardless of row order and never falls through to `offline_security`
for a mapped label.

### Property 4: Non-trade and API flows unchanged
Only `TradeImport#mapping_steps` adds the step. Non-trade imports and the API import path
retain existing behavior.

## Testing Strategy

Minitest + fixtures (no RSpec behavioral tests), per `AGENTS.md`/testing guide:

- `test/models/import/security_mapping_test.rb` — `mappables_by_key` auto-matches a seeded
  security by ticker; leaves an unknown label unmatched; `create_mappable!` resolves a
  combobox `value` to a `Security`; `values_count` counts rows by ticker.
- `test/models/trade_import_test.rb` — add cases: a clean-ticker CSV publishes trades with
  no required interaction; a descriptive-label CSV with a user-mapped security publishes a
  trade referencing the mapped security and NOT an offline security; `mapping_steps`
  includes `Import::SecurityMapping` for trade imports and excludes it for other types.
- `test/controllers/import/mappings_controller_test.rb` — updating a security mapping with a
  combobox `value` resolves and assigns the mappable.
- Regression: an existing non-trade mapping test still passes (scope/`mapping_steps` guard).

## Decisions and tradeoffs

- **Local-DB auto-match, provider search only on demand** (user-confirmed): a brand-new
  ticker not yet in the instance's DB will show as unresolved the first time and be
  confirmed once via the combobox. Tradeoff accepted for speed and rate-limit safety.
- **No `CREATE_NEW_KEY` path:** securities are global and the resolver already creates
  offline records; a "create new" option would duplicate that and risk bad tickers.
- **Resolve-on-update** (vs. resolve-at-publish): chosen so the confirm UI shows matched
  state immediately; cost is a narrow, type-guarded change to `MappingsController#update`.

## Resolved pre-task checks

- **No migration needed.** `import_mappings` (STI, uuid pk) already has `key`, `value`,
  `mappable_type`, `mappable_id`, `create_when_empty` — the security mapping reuses them.
- **Combobox reuse confirmed.** The trade form uses the built-in form helper
  `form.combobox :ticker, securities_path(country_code: Current.family.country),
  name_when_new: "model[manual_ticker]", ...` (gated on `Security.providers.any?`, with a
  plain `text_field :manual_ticker` fallback when no provider is configured). The mapping
  `_form.html.erb` security branch reuses this same `form.combobox` helper pointed at
  `securities_path`, with the equivalent no-provider `manual_ticker`-style fallback. The
  submitted combobox id flows into the mapping's `value`.
