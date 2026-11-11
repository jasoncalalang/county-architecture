# ADR 0009: Flyway migrates at API start; migrations stay compatible with the previous release

Date: 2026-10-31
Status: accepted

## Context

Schema changes have to reach the cluster together with the code that needs
them. Flyway can run when the application starts, as a Helm pre-upgrade hook
Job, as an init container, or by hand. The api Deployment rolls with
`maxUnavailable: 0`, so the old pod serves while the new one starts.

Alternatives considered: a pre-upgrade Job, which gives migrations their own
logs and lifecycle but needs a migrate-only entrypoint (a command-line mode
in api, or the Flyway image with SQL mounted) and a hook ordering story; an
init container, which runs the same code per pod and adds nothing over
migrating at start; manual migration, which puts a person in every release.

## Decision

- `quarkus.flyway.migrate-at-start=true` in api. Flyway's own lock
  serialises concurrent pods.
- Rules for every migration:
  1. Forward only. No undo scripts.
  2. The schema after the migration must work for the previous release:
     expand first (nullable columns, defaults, new tables), switch code,
     contract in a later release.
  3. It finishes inside the api startup probe window, 60 seconds.
  4. It runs in CI against a fresh schema on every build (ADR 0005), so an
     unrunnable migration never reaches the cluster.

## Consequences

- A failing migration shows as a pod that never becomes ready. `--atomic`
  rolls the Deployment back and the previous release keeps running on the
  partly migrated schema, which rule 2 makes safe. `helm rollback` is safe
  for the same reason.
- The api database role owns the schema and holds DDL rights. Fine with one
  role in v1.
- Revisit, moving to a Job, when a migration needs more than the startup
  window or when DDL rights must be split from the application role.
