# ADR 0013: Explicit SQL for the ordering persistence boundary

Date: 2026-11-15
Status: proposed

## Context

[ADR 0001](0001-quarkus-api-nextjs-web.md) selects Quarkus, Hibernate ORM
with Panache, Flyway and Postgres. The first backend implementation in
[county PR #2](https://github.com/jasoncalalang/county/pull/2) uses
parameterised JDBC statements through Quarkus's Agroal datasource instead
of Panache entity operations. This record makes that deviation explicit for
review; implementation in a draft PR is not acceptance of this ADR.

The ordering write path needs several operations to succeed or roll back
together: verify an active reseller token, resolve a repeated submission,
read a consistent product-price snapshot, allocate the next business order
number, save immutable lines, and mark the reseller active. The proposed
[functional specification](../design/2026-10-31-v1-functional-spec.md)
requires order numbers without rollback gaps and idempotency keys unique
per business. [ADR 0012](0012-orders-are-immutable-after-placement.md)
already requires the saved order to retain its placement-time values.

Explicit SQL makes the lock targets, tenant predicates and atomic counter
update visible together in a small persistence layer. Panache could express
these operations too, with pessimistic locking or native queries; the choice
is a simplicity tradeoff, not a claim that ORM cannot provide correctness.

## Proposed decision

- Permit explicit, parameterised SQL for the v1 backend persistence boundary,
  using the managed Agroal datasource and a single JDBC connection for each
  transaction. Keep database details out of the pure domain rules and HTTP
  response contracts.
- Make the transaction helper commit only after all work succeeds and roll
  back on failure. Do not mix independently managed JDBC and ORM transactions
  in one business operation.
- Keep schema ownership with forward-only Flyway migrations, as required by
  [ADR 0009](0009-flyway-migrates-at-api-start.md). Do not use ORM schema
  generation in place of migrations.
- For placement, lock the active reseller row, then the business row, then
  read the requested product rows under shared locks in ascending product ID
  order. Keep the business counter update, order, lines and reseller update
  in that transaction. Product edits take an exclusive product lock. Future
  operations acquiring multiple locks must preserve a compatible order.
- Check the business-scoped idempotency key while holding the business lock.
  A replay returns the original order only for the same reseller; another
  reseller using that key must not receive its details. Keep a unique
  database constraint as a second line of defence.
- Use an ordinary transactional business counter for visible order numbers,
  not a PostgreSQL sequence: a rolled-back placement must not consume a
  visible number. Internal database IDs may still use sequences.
- Enforce tenant scope in queries and composite foreign keys. Derive that
  scope from the authenticated owner or reseller credential, never a supplied
  tenant ID. An inaccessible resource remains a 404.

If accepted, this is a scoped exception to ADR 0001's Panache persistence
choice. The rest of ADR 0001 remains unchanged: Java 21, Quarkus, Postgres,
OpenAPI and the Next.js frontend contract. It neither requires removing
Panache nor commits future modules to JDBC.

## Alternatives considered

- **Panache entities with pessimistic locks or native queries:** consistent
  with ADR 0001 and a reasonable future choice. For this small slice it adds
  entity/state mapping while the important write operations remain explicit
  SQL or lock-sensitive queries.
- **Optimistic retries:** potentially less serialisation, but retries must
  revalidate token state, idempotency, settings and price snapshots. Defer
  that complexity until measured contention justifies it.
- **Database sequence for visible order numbers:** simple and concurrent,
  but rollbacks consume sequence values, conflicting with the proposed
  no-gap requirement. Revisit only if that product requirement changes.

## Consequences

- The transaction and tenant boundary are easy to inspect, but SQL, row
  mapping, constraints and migrations must be maintained deliberately.
  Parameter binding is mandatory; user input must not become SQL syntax.
- A business-row lock serialises placements within that business. This is
  an intentional v1 throughput tradeoff, not a scalability guarantee. Keep
  transactions short and perform no network calls while holding locks.
- Adding a new workflow requires reviewing its lock order, rollback behaviour
  and tenant predicates. Database constraints complement, rather than
  replace, API authorisation and tests.
- Tests use real PostgreSQL supplied externally, preserving
  [ADR 0005](0005-tests-run-without-docker.md). Verification must cover
  concurrent same-key submissions, cross-reseller key collisions, rollback
  without a number gap, isolation and product-update/placement interleavings.
- Typed API DTOs and generated OpenAPI contracts remain required. Internal
  SQL row maps are not a replacement for the public response schema.
- This decision does not change authentication. Google ID-token verification
  and a separate `(provider, subject)` identity mapped to a user remain the
  accepted design in [ADR 0002](0002-google-sign-in-for-wholesalers.md).
  Direct ownership by a Google subject in an early foundation is not a new
  identity decision and does not replace that user/identity model.

## Review and revisit

Accept or reject this exception explicitly during backend review. Revisit
if measured per-business contention affects order submission, persistence
mapping becomes more complicated than the SQL it serves, or a future module
needs ORM behaviour. Until accepted, ADR 0001 remains the accepted baseline.
