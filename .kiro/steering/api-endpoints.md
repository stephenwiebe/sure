---
inclusion: fileMatch
fileMatchPattern: ['app/controllers/api/**', 'test/controllers/api/**', 'spec/requests/api/**', 'spec/swagger_helper.rb', 'docs/api/**']
---

# API v1 endpoints

Full guide: [API endpoint consistency](../../docs/llm-guides/api-endpoint-consistency.md).

## Behavioral tests (Minitest)

- Location: `test/controllers/api/v1/{resource}_controller_test.rb`.
- All new/changed actions must have Minitest behavioral coverage.
- Use `ApiKey.create!` with `api_headers(api_key)` →
  `{ "X-Api-Key" => api_key.display_key }`. No OAuth/Bearer in these tests.
- Cover: index/show (create/update/destroy for write endpoints), read-only key
  blocking writes (403), invalid params (422), invalid date (422), not found (404),
  missing auth (401).

## Documentation specs (rswag)

- Location: `spec/requests/api/v1/{resource}_spec.rb`.
- **Docs only.** No `expect(...)` or `assert_*`. `run_test!` without assertion blocks.
- Every spec must use the shared API-key auth pattern (no Doorkeeper/OAuth).
- Reusable schemas in `spec/swagger_helper.rb`.
- Regenerate after edits: `RAILS_ENV=test bundle exec rake rswag:specs:swaggerize`.

## Runtime

- The [API base controller](../../app/controllers/api/v1/base_controller.rb) sets
  current context, skips CSRF and applies API-key rate limits.
- Both OAuth (Doorkeeper) and `X-Api-Key` are supported at runtime — the API-key-only
  requirement is for the test/doc patterns.
- Preserve existing authentication, scope, and strong-parameter behavior.
