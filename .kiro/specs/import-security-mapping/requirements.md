# Requirements: Import Security Mapping

## Introduction

When importing investment trades via CSV (`TradeImport`), the `ticker` column is resolved
directly through `Security::Resolver`, which treats the value as a stock/fund ticker
symbol. Brokerage statements frequently label positions with a descriptive name
(e.g. `VANGUARD TOTAL BOND ETF MARKET`) rather than a ticker (`BND`). Today such a value
falls through every resolver path to `offline_security`, which creates a bogus security
whose ticker is the full descriptive string, marked `offline` so no prices are ever
fetched. The import appears to succeed but produces an unpriced, mislabeled holding.

This feature adds a security-mapping step to the web `TradeImport` flow, mirroring the
existing `Import::AccountMapping` / `Import::CategoryMapping` pattern. Ticker values that
already resolve to a real security are auto-matched and require no user action; only
unresolved labels are surfaced for the user to map to a real ticker via the existing
security-search combobox.

## Scope

- **In scope:** Web UI `TradeImport` CSV flow (upload → configure → map → confirm → clean).
- **Out of scope:** The API import flow (`Api::V1::ImportsController`), which keeps its
  current `Security::Resolver` fallback behavior unchanged. Provider-connected investment
  paths (Plaid, SimpleFIN, SnapTrade, etc.) resolve securities from provider data and are
  unaffected.

## Requirements

### Requirement 1: Security mapping step for trade imports

**User Story:** As a user importing a trade CSV, I want unrecognized ticker labels to be
collected into a mapping step, so that I can map each distinct label to a real security
before the import is published.

#### Acceptance Criteria

1. WHEN a `TradeImport` has uploaded rows THEN the system SHALL include
   `Import::SecurityMapping` in the import's `mapping_steps`.
2. WHEN the mapping step is built THEN the system SHALL create one mapping keyed by each
   distinct `ticker` value across the import's rows.
3. WHEN a row's `ticker` value resolves to an existing `Security` in the family-agnostic
   securities table (exact ticker + exchange match) THEN the system SHALL auto-match that
   mapping to the resolved security without requiring user interaction.
4. WHEN a row's `ticker` value does not resolve to an existing security THEN the system
   SHALL present that mapping as unresolved and require the user to select a security.
5. WHEN all security mappings are resolved (auto-matched or user-selected) THEN the import
   SHALL be publishable with respect to security mapping.

### Requirement 2: Security selection via existing search

**User Story:** As a user mapping an unrecognized label, I want to search for the correct
security by name or ticker, so that I can pick the right instrument and price provider.

#### Acceptance Criteria

1. WHEN the user opens the selector for an unresolved security mapping THEN the system
   SHALL present the existing security-search combobox used by the manual trade form.
2. WHEN the user selects a search result THEN the system SHALL persist the chosen ticker,
   exchange operating MIC, and price provider for that mapping.
3. WHEN a mapping selection is submitted THEN the system SHALL resolve it to a persisted
   `Security` record via `Security::Resolver` using the chosen ticker and exchange.
4. IF the user has no enabled securities provider THEN the system SHALL still allow mapping
   to an existing or offline security, consistent with current resolver behavior.

### Requirement 3: Import uses mapped securities

**User Story:** As a user, I want the published trades to reference the securities I mapped,
so that my holdings carry the correct ticker and fetch real prices.

#### Acceptance Criteria

1. WHEN `TradeImport#import!` runs THEN for each row the system SHALL use the security from
   that row's resolved `Import::SecurityMapping` rather than resolving `row.ticker` directly.
2. WHEN a row's label was auto-matched THEN the resulting `Trade` SHALL reference the same
   security the resolver would have produced for the clean ticker, preserving current
   behavior for CSVs that already use real tickers.
3. WHEN a row's label was user-mapped THEN the resulting `Trade` SHALL reference the
   user-selected security, and that security SHALL NOT be marked `offline` solely because
   the original label was descriptive.
4. WHEN the import is reverted THEN security mappings SHALL be cleaned up with the import,
   consistent with other mapping types (`dependent: :destroy`).

### Requirement 4: Non-regression for existing flows

**User Story:** As an existing user, I want clean trade CSVs and all non-trade imports to
behave exactly as before, so that this feature introduces no friction or breakage.

#### Acceptance Criteria

1. WHEN a `TradeImport` CSV uses only real, resolvable tickers THEN the user SHALL NOT be
   required to interact with the security mapping step (all auto-matched).
2. WHEN any non-`TradeImport` import runs THEN the system SHALL NOT add a security mapping
   step and SHALL behave exactly as before.
3. WHEN a trade import is created or populated via the API THEN the system SHALL retain the
   current direct-resolver behavior and SHALL NOT require an interactive mapping step.
4. WHEN the `Import::Mapping` scopes are queried THEN a `securities` scope SHALL return only
   `Import::SecurityMapping` records, consistent with the existing `accounts`/`categories`/
   `tags` scopes.
