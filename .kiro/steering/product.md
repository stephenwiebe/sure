---
inclusion: always
---

# Product

Sure is a self-hostable personal finance and wealth management app — a community
fork of the archived Maybe Finance project. It is **not affiliated with or endorsed
by Maybe Finance Inc.** ("Maybe" is their trademark; this fork is "Sure").

Users manage accounts, balances, transactions, trades, holdings, budgets, goals and
multi-currency reporting. Data arrives through connected providers (Plaid, SimpleFIN,
Monobank, Lunchflow and others) or manual/CSV import. Optional AI assistant features
are backed by OpenAI/Anthropic.

## Runtime modes

The app runs in one of two modes via `Rails.application.config.app_mode`, derived
from `SELF_HOSTED`/`SELF_HOSTING_ENABLED`:

- `managed` — the hosted multi-tenant deployment.
- `self_hosted` — a single self-hosted install.

Provider availability and features can differ between modes. Never assume a provider
or feature is present; gate on configuration.

## Access surfaces

Sure is reached from the browser, a macOS desktop app (`desktop/`, Tauri), a mobile
app (`mobile/`, Flutter), API clients, and LLM agents. See `docs/clients.md`.

## License

AGPLv3. Never introduce code or dependencies incompatible with AGPLv3.
