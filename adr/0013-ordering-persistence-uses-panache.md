# ADR 0013: Ordering persistence uses Panache with pessimistic locks

Date: 2026-11-17
Status: accepted

## Context

[ADR 0001](0001-quarkus-api-nextjs-web.md) chose Hibernate ORM with Panache
for persistence. The first backend, merged in
[county PR #2](https://github.com/jasoncalalang/county/pull/2), used
parameterised JDBC statements instead, and a proposed ADR 0013 in
[county-architecture PR #2](https://github.com/jasoncalalang/county-architecture/pull/2)
asked to accept that as an exception for the ordering write path.

Placing an order has to succeed or roll back as one unit. It verifies an
active reseller token, resolves a repeated submission, reads a consistent
product price snapshot, takes the business's next order number, saves
immutable lines ([ADR 0012](0012-orders-are-immutable-after-placement.md))
and marks the reseller active. The
[functional spec](../design/2026-10-31-v1-functional-spec.md) requires order
numbers without gaps and idempotency keys unique per business. The proposal
itself noted that Panache can express all of this with pessimistic locks.

Alternatives considered: the explicit-SQL exception, which keeps lock
targets visible in the SQL text but splits the codebase between two
persistence styles and maps rows by hand; optimistic locking with retries,
which serialises less but has to revalidate the token, key, settings and
prices on every retry; a database sequence for order numbers, which is
simple but consumes numbers on rollback.

## Decision

- All backend persistence uses Hibernate ORM with Panache entities over the
  Flyway-owned schema. There is no separate JDBC persistence layer.
  Hibernate never generates or alters the schema
  ([ADR 0009](0009-flyway-migrates-at-api-start.md)).
- Each business operation is one `@Transactional` service method. Response
  bodies are typed view records built from entities; entities are never
  serialised.
- Locks use JPA pessimistic lock modes, always in this order: reseller,
  business, products by ascending id, order. Placement locks the reseller
  and the business for writing and reads products under a shared lock;
  a product edit locks the product for writing. A new workflow that takes
  more than one lock keeps this order.
- The idempotency key is checked while the business lock is held. A replay
  returns the original order only to the reseller who placed it; another
  reseller of the same business gets a conflict. A unique constraint stays
  as the second line of defence.
- Visible order numbers come from a counter on the business row, updated in
  the placement transaction, so a rolled-back placement consumes no number.
  Row ids may use identity columns.
- A PostgreSQL feature with no JPA equivalent, such as the advisory lock
  that serialises a first sign-in, runs as a native query on the same
  entity manager and transaction, never on a separate connection.
- Tenant scope is in every query and in composite foreign keys. It comes
  from the signed-in owner or the reseller token, never from the request.
  A resource in another tenant is a 404.

This replaces the proposed explicit-SQL ADR 0013, which was not accepted.

## Consequences

- One persistence style across the backend, as ADR 0001 intended.
- Lock intent is in the lock mode on each query rather than in SQL text, so
  reviewers check the lock order in the service code. Tests that wait on a
  real lock match Hibernate's generated SQL, not a hand-written statement.
- The business-row lock serialises placements within one business. That is
  an accepted v1 limit, not a scalability guarantee: keep these
  transactions short and make no network calls while holding locks.
- Constraint violations surface as Hibernate exceptions. The API maps a
  unique violation to 409 and does not log the SQL error, because its
  detail can contain a mobile number or an order-link token.
- Tests keep using an external PostgreSQL
  ([ADR 0005](0005-tests-run-without-docker.md)) and must cover concurrent
  same-key submissions, keys reused across resellers, rollback without a
  number gap, tenant isolation, and product edits racing a placement.
- Revisit if measured contention on order placement in one business becomes
  a problem; optimistic locking with retries is the first alternative.
