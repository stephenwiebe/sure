---
inclusion: fileMatch
fileMatchPattern: ['app/models/provider/**', 'app/models/*_item/**', 'app/models/*_entry/**', 'app/models/account/provider_import_adapter.rb', 'app/models/exchange_rate/**', 'app/models/security/**']
---

# Provider and sync integration

Full guides:
- [Provider sync guidance](../../docs/llm-guides/providers.md)
- [Adding a securities provider](../../docs/llm-guides/adding-a-securities-provider.md)
- [Adding a classification provider](../../docs/llm-guides/adding-a-classification-provider.md)

## Key patterns

- Interchangeable provider concepts are registered at runtime through
  `Provider::Registry` and `Setting`.
- Concept providers inherit from `Provider` and use `with_provider_response` to return
  `Provider::Response` (`success?`, `data`, `error`). Raise inside the wrapper when
  valid data cannot be produced.
- Domain models select providers through their `Provided` concerns, not the registry.
- Use `DebugLogEntry.capture(...)` for support-relevant failures and partial responses.
  Include `category`, `level`, `message`, `source`, `provider_key` and `metadata`.
  Attach `family` and `account_provider` when available.

## Pending transactions

- Store provider metadata on `Transaction#extra` under the provider namespace.
- `import_transaction` deep-merges; use `replace_extra_namespaces: ["<key>"]` when
  the provider owns its namespace outright.
- `Transaction#pending?` and pending scopes share `PENDING_PROVIDERS`.

## FX metadata

- `extra["<provider>"]["fx_from"]` when transaction and account currencies differ.
- Entries take currency from the account, not the operation currency.
