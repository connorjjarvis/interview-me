> Lives at ~/.review-me-senior/<repo-name>/rules.md — local, per-user, out of your repo.

# Learned rules — quote-service

### Service methods return `Result<T>` instead of throwing

- **Verdict:** accepted
- **Scope:** `src/services/**`
- **Rationale:** The service layer deliberately models expected failures (not-found, validation) as values so callers must handle them explicitly instead of relying on try/catch; this is a project-wide convention, not a shortcut someone took.
- **Added:** 2026-07-07

### New endpoints must have an integration test

- **Verdict:** expected
- **Scope:** category: tests
- **Rationale:** Two production incidents in Q1 2026 traced back to endpoints that only had unit tests against a mocked pipeline and never exercised real routing/auth end to end.
- **Added:** 2026-07-07

### Raw SQL string concatenation

- **Verdict:** forbidden
- **Scope:** category: security
- **Rationale:** String-concatenated SQL is an injection vector; the repo standardizes on parameterized queries via Dapper's `QueryAsync<T>(sql, parameters)`.
- **Added:** 2026-07-07

### Cache keys for tenant-scoped data must include `tenantId`

- **Verdict:** expected
- **Scope:** `src/cache/**`
- **Rationale:** Omitting `tenantId` from a cache key lets one tenant's cached data leak into another tenant's response; added after a cross-tenant cache-poisoning bug surfaced during the response-cache-layer rollout.
- **Added:** 2026-07-07
