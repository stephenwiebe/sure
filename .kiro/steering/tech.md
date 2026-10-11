---
inclusion: always
---

# Technology

Canonical agent guidance lives in [`AGENTS.md`](../../AGENTS.md) and the task guides
under [`docs/llm-guides/`](../../docs/llm-guides/README.md). Read those for domain
and convention detail; this file is the quick-orientation layer. Read the real
toolchain from the manifests below, not from memory — versions drift.

## Stack

- **Ruby on Rails 8.1** backend. Ruby version is in [`.ruby-version`](../../.ruby-version)
  (`mise.toml` pins it alongside Node). Gems in [`Gemfile`](../../Gemfile).
- **PostgreSQL** (`pg` gem) is the database; schema in `db/schema.rb`.
- **Redis** + **Sidekiq** (with `sidekiq-cron`, `sidekiq-unique-jobs`) for background jobs.
- **Hotwire** (Turbo + Stimulus), **ViewComponent**, **importmap-rails**, **Propshaft**.
- **Tailwind CSS v4** via `tailwindcss-rails` / `@tailwindcss/cli`.
- **Puma** web server.
- Auth: **Doorkeeper** (OAuth), **Pundit** (authorization), **OmniAuth** (OIDC/SAML/
  Google/GitHub), `bcrypt`, `webauthn`, `rotp`/`rqrcode` (TOTP).
- Monitoring: Sentry, PostHog, Logtail, Skylight, `rack-mini-profiler`, `vernier`.
- JavaScript lives in `app/javascript/`; linted/formatted by **Biome**
  ([`biome.json`](../../biome.json)) — double quotes, organize imports.

## Commands

These are references, not permission to run them. Do not start servers, touch
`tmp/restart.txt`, run `rails credentials`, or run migrations/`bin/setup` unless
the environment work is explicitly requested. Full detail:
[development guide](../../docs/llm-guides/development.md).

| Task | Command |
| --- | --- |
| Dev processes (Rails, Sidekiq, Tailwind) | `bin/dev` |
| Rails console | `bin/rails console` |
| Full test suite | `bin/rails test` |
| One file / line | `bin/rails test test/models/account_test.rb[:42]` |
| System tests | `DISABLE_PARALLELIZATION=true bin/rails test:system` |
| Ruby lint (+ safe autocorrect) | `bin/rubocop` / `bin/rubocop -a` |
| ERB lint | `bundle exec erb_lint ./app/**/*.erb -a` |
| JS lint / fix | `npm run lint` / `npm run lint:fix` |
| Biome format check / fix | `npm run format:check` / `npm run format` |
| Security scan | `bin/brakeman --no-pager` |
| Regenerate OpenAPI docs | `RAILS_ENV=test bundle exec rake rswag:specs:swaggerize` |

## Before opening a PR

Run the full [pre-PR checklist](../../docs/llm-guides/development.md#before-opening-a-pull-request):
full `bin/rails test`, applicable system tests, `bin/rubocop`, ERB lint, `npm run lint`,
and `bin/brakeman`. All required checks must pass before creating the PR. CI additionally
runs a Pipelock security scan on every PR to `main`.

## Testing

Minitest + fixtures for behavioral tests (Mocha for mocks, VCR for external calls).
RSpec/rswag is **documentation-only** for OpenAPI. Do not add RSpec behavioral tests
or factories. See the [testing guide](../../docs/llm-guides/testing.md).

## Dependencies

Prefer built-in Rails patterns. A new dependency needs a strong technical or business
reason. Pin versions. Never introduce AGPLv3-incompatible dependencies. Never commit
secrets — use environment variables and `.env.local`.
