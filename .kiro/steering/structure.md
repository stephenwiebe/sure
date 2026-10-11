---
inclusion: always
---

# Structure

```
sure/
├── app/
│   ├── assets/            # Tailwind CSS, builds, images
│   ├── channels/          # Action Cable
│   ├── components/        # ViewComponents (DS::* design system)
│   │   └── DS/            # Design system primitives
│   ├── constraints/       # Route constraints
│   ├── controllers/       # Web + API (api/v1/) controllers
│   ├── data_migrations/   # Data migration classes
│   ├── helpers/           # View helpers (icon helper lives here)
│   ├── javascript/        # Stimulus controllers, Hotwire JS
│   ├── jobs/              # Sidekiq jobs (SyncJob, ImportJob, etc.)
│   ├── mailers/           # Action Mailer
│   ├── middleware/         # Rack middleware (OmniAuth error handler)
│   ├── models/            # Domain models, concerns, POROs
│   │   ├── concerns/      # Accountable, Syncable, Entryable, etc.
│   │   └── provider/      # Registry, concepts, provider implementations
│   ├── policies/          # Pundit policies
│   ├── services/          # Service objects (limited use)
│   └── views/             # ERB templates
├── bin/                   # Scripts (setup, dev, rails, rubocop, brakeman)
├── config/
│   ├── environments/      # Rails env configs
│   ├── initializers/      # App initializers
│   ├── locales/           # I18n YAML files
│   ├── routes.rb          # Route definitions
│   ├── schedule.yml       # Sidekiq-cron schedule
│   └── sidekiq.yml        # Sidekiq config
├── db/
│   ├── migrate/           # Database migrations
│   ├── schema.rb          # Canonical schema (schema_format = :ruby)
│   └── seeds/             # Seed data
├── docs/
│   ├── api/               # OpenAPI spec (generated)
│   ├── hosting/           # Self-hosting guides
│   ├── llm-guides/        # Agent/developer guides (the real source of truth)
│   └── onboarding/
├── lib/                   # Shared libraries (Money, Semver, feature flags)
├── spec/                  # rswag-only: OpenAPI docs specs (NO behavioral tests)
├── test/                  # Minitest: models, controllers, components, etc.
│   ├── fixtures/          # Test fixtures
│   ├── support/           # Test helpers
│   └── vcr_cassettes/     # VCR recordings
├── desktop/               # Tauri macOS desktop app
├── mobile/                # Flutter mobile app
├── charts/                # Helm chart
└── design/                # Design tokens
```

## Key conventions

- Business logic lives in `app/models/` via POROs and concerns. Thin controllers.
- Use `Current.user` and `Current.family`, never `current_user` / `current_family`.
- Tests mirror `app/` under `test/`. Fixtures in `test/fixtures/`.
- Migrations in `db/migrate/`; use the current Rails migration version.
- API endpoints in `app/controllers/api/v1/`; behavioral tests in
  `test/controllers/api/v1/`, docs-only rswag in `spec/requests/api/v1/`.
