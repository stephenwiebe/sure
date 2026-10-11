# Tasks: Import Security Mapping

Each task is independently implementable and verifiable with `bin/rails test`. No DB
migration is required (the `import_mappings` STI table already carries `value`). Scope is
web `TradeImport`; the API flow and non-trade imports stay unchanged.

- [ ] 1. Add `Import::SecurityMapping` model and `securities` scope
  - Create `app/models/import/security_mapping.rb` as a subclass of `Import::Mapping`.
  - Implement `self.mappables_by_key(import)` to key off distinct `import.rows.map(&:ticker)`
    and auto-match via local DB lookup (`Security.find_by_ticker_and_exchange`), no network.
  - Implement `requires_selection?`, `values_count` (count rows by `ticker`),
    `mappable_class` (`Security`), and `create_mappable!` (parse `value` via
    `Security.parse_combobox_id`, resolve with `Security::Resolver`, assign mappable).
  - Add `scope :securities, -> { where(type: "Import::SecurityMapping") }` to
    `Import::Mapping`.
  - _Requirements: 1.2, 1.3, 1.4, 2.2, 2.3, 4.4_

- [ ] 2. Unit tests for `Import::SecurityMapping`
  - `test/models/import/security_mapping_test.rb`: auto-matches a seeded security by ticker;
    leaves an unknown descriptive label unmatched; `create_mappable!` resolves a combobox
    `value` to a `Security`; `values_count` counts rows by ticker.
  - _Requirements: 1.2, 1.3, 1.4, 2.3_

- [ ] 3. Wire the step into `TradeImport`
  - Add `Import::SecurityMapping` to `TradeImport#mapping_steps`.
  - In `import!`, resolve each row's security via
    `mappings.securities.mappable_for(row.ticker)` with a fallback to the existing
    `find_or_create_security(...)` so auto-matched/edge rows still resolve.
  - _Requirements: 1.1, 3.1, 3.2, 3.3, 4.2_

- [ ] 4. `TradeImport` behavioral tests
  - `test/models/trade_import_test.rb`: a clean-ticker CSV publishes trades with no required
    interaction (all auto-matched); a descriptive-label CSV with a user-mapped security
    publishes a trade referencing the mapped security and NOT an offline one;
    `mapping_steps` includes `Import::SecurityMapping` for trade imports and excludes it for
    non-trade imports.
  - _Requirements: 1.1, 1.5, 3.1, 3.2, 3.3, 4.1, 4.2_

- [ ] 5. Resolve-on-update in `Import::MappingsController`
  - For `Import::SecurityMapping`, after saving `value`, resolve and assign the mappable so
    the confirm screen reflects a matched state immediately. Keep the change type-guarded so
    other mapping types are unaffected.
  - `test/controllers/import/mappings_controller_test.rb`: updating a security mapping with a
    combobox `value` resolves and assigns the mappable; updating a non-security mapping is
    unchanged.
  - _Requirements: 2.1, 2.2, 2.3_

- [ ] 6. Mapping form view: security combobox branch
  - Add a `Import::SecurityMapping` branch to `app/views/import/mappings/_form.html.erb`
    that renders `form.combobox` pointed at `securities_path` (reusing the trade form's
    helper), with the no-provider `manual_ticker`-style fallback when
    `Security.providers.none?`. Auto-matched mappings show the matched security as current
    value; unresolved show the searchable control.
  - Follow the design system: DS primitives, `icon` helper, `t()` for strings, functional
    tokens only.
  - _Requirements: 2.1, 2.4_

- [ ] 7. i18n and confirm-page labels
  - Add locale keys for the security step title/description
    (`import_security_mapping_title` / `_description`) and the confirm `_mappings` CSV/Sure
    column labels for securities. Ensure `mapping_label` renders correctly for the new type.
  - _Requirements: 1.1, 2.1_

- [ ]* 8. System test for the full web trade-import mapping flow
  - `DISABLE_PARALLELIZATION=true bin/rails test:system`: upload a trade CSV containing one
    clean ticker and one descriptive label, confirm only the descriptive label requires
    selection, map it via the combobox, and verify the published trade references the mapped
    security. Optional because the behavioral + controller tests cover the logic; this
    guards the end-to-end UI wiring.
  - _Requirements: 1.4, 1.5, 2.1, 2.2, 3.2_

- [ ] 9. Pre-PR verification
  - Run the pre-PR checklist: `bin/rails test`, applicable system tests, `bin/rubocop`, ERB
    lint, `npm run lint`, `bin/brakeman`. Confirm non-trade imports and the API import path
    are unaffected.
  - _Requirements: 4.1, 4.2, 4.3_
