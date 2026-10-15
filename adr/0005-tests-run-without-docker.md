# ADR 0005: The test suite must run without Docker

Date: 2026-10-16
Status: accepted

## Context

Quarkus Dev Services start a database container for integration tests. The
team's development host runs Kubernetes with containerd and has no Docker
daemon, which in a previous project left most of the backend test suite
unrunnable on the machine where development happens.

## Decision

- Domain rules (delivery dates, cut-off, transitions, totals, token checks)
  are plain classes tested with JUnit, with no framework injection.
- API tests use `@QuarkusTest` against a Postgres given by
  `COUNTY_TEST_JDBC_URL`. Dev Services are disabled in the test profile.
  Flyway migrates a fresh schema per run.
- CI provides a throwaway Postgres; a developer points the variable at any
  Postgres they have.
- Tenant-isolation tests (tenant B gets 404 on tenant A's resources) are
  required for every tenant-scoped endpoint and gate merges.

## Consequences

- Developers must have a Postgres reachable; the API `README` documents how.
- No container runtime is needed to run the full suite anywhere.
- Test data setup is explicit per test class rather than implied by a
  container lifecycle.
