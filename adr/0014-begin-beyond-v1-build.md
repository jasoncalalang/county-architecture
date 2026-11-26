# ADR 0014: Begin the Beyond-v1 build, pricing first

Date: 2026-11-26
Status: accepted

## Context

county v1 is a demo, not a pilot ([v1 design](../design/2026-10-16-v1-design.md)).
Its scope was drawn as the thinnest slice that runs the wholesaler→reseller
loop in front of a prospect. The
[functional spec](../design/2026-10-31-v1-functional-spec.md) §2 lists the v1
features as exhaustive and names, in §2 and §12 *Beyond v1*, the features
deferred by decision: price tiers and per-reseller price lists, minimum order
quantities, payments beyond the `paid` flag, and wholesaler edits / short-
delivery discrepancy reports.
[ADR 0012](0012-orders-are-immutable-after-placement.md) defers discrepancy
reports specifically ("Short deliveries ... are a discrepancy report,
deferred past v1") and asks to "revisit ... when a pilot customer's short
deliveries need recording."

A reverse-engineering study of the Itemcount ERP
(`itemcount-deep-rea.md` §5.2 "What county should build next") ranks five
post-v1 increments, all of which are faithful minimal analogues of Itemcount
capabilities and all of which fit county's existing conventions: REST not
GraphQL, Flyway-owned schema with composite `(tenant_id, id)` uniqueness and
tenant-scoped FKs, the ADR 0013 pessimistic lock order, typed view/input
DTOs, and immutability after placement. The design amendment
[Beyond-v1 build plan informed by Itemcount](../design/2026-11-23-beyond-v1-itemcount-build-plan.md)
works each one through to a migration, endpoints,
DTOs, lock-order impact and tests.

We need a decision to lift the v1 scope freeze for these five, in a stated
order, without reopening the decisions v1 made for good reasons.

## Decision

- Accept the five features in the amendment — per-reseller price tiers, quotes
  → order, receive-with-discrepancy, MOQ + order-create windows, and online
  payment link — as approved post-v1 work, to be built in that priority order.
  Price tiers go first; quotes build on the captured-price primitive tiers
  introduce.
- Each feature is implemented as its own `V3+__*.sql` migration under ADR 0009
  rules (forward-only, expand-first, no-op for existing tenants), with new
  root tables carrying `UNIQUE (tenant_id, id)` and every table carrying
  tenant-scoped composite FKs.
- The ADR 0013 lock order is extended, not broken. ADR 0013 is accepted and
  is not edited. A new ADR
  ([ADR 0015](0015-added-lock-targets-follow-the-canonical-four.md)),
  accepted before the price-tier code merges, supersedes its lock-order
  bullet with a rule rather than a longer list:
  - reseller, business, products by ascending id and order keep their order;
  - a workflow may lock a row outside those four only after every one of
    the four it locks, and no workflow locks such a row before one of them;
    a row the workflow inserts in its own transaction is not a lock target;
  - the added lock targets are listed, in order, in the lock note of
    `api/README.md` in the county repository, and a workflow that takes
    more than one takes them in that order. The list is updated in the
    change that adds a target.
- The first added target is the `price_tier` row, locked after the reseller
  when a reseller's tier is assigned. Placement takes no new lock for tiers,
  because a product's tier prices are guarded by that product's lock.
  Quotes (reseller → business → quote, then the new order) and payments
  (the `payment_link` row after the order) fit the rule as designed; each
  adds its line to the list when it is built and needs no further ADR
  unless it cannot fit the rule. Discrepancy hangs children off the order
  lock without locking them; MOQ/windows add no lock.
- ADR 0012 stands. No edit endpoint is added. Discrepancy reports are an
  append-only sibling record, not an order edit; a short delivery never
  recomputes an order total. Tier/quote/payment changes never mutate captured
  `order_line` prices; only `status` and `paid` ever change on a placed order.
- Two guard relaxations require explicit owner sign-off before their code
  merges, because each loosens an accepted rule: (a) the override `DELETE` in
  price tiers is the first delete in the codebase and is justified only as
  pricing config with no history; (b) collectible payments let the webhook set
  `paid` on a placed or confirmed order, loosening `mark-paid`'s current
  "delivered only" guard and requiring `deliver` to stop overwriting the
  `paid` flag of an order that is already paid.
- What deactivating a price tier does to resellers already assigned to it is
  decided by the owner before the tier code merges; the reference
  implementation leaves them on the tier's prices.
- Each feature ships tests first, covering the matrix ADR 0013 already
  mandates (concurrent same-key submissions, keys reused across resellers,
  rollback without a number gap, tenant isolation, edits racing a placement)
  plus its own immutability and resolution rules, and must pass the OpenAPI
  drift check.

## Consequences

- The v1 scope freeze is lifted for exactly these five, in this order. The
  functional spec §12 rows for price tiers, MOQ, payments and discrepancy
  reports move from *Beyond v1* to in-progress as each lands. The functional
  spec (§12, and §11 for accepted amendments) is updated at that point; this
  ADR stays as accepted, and a later ADR supersedes it if the decision
  changes.
- Quotes have the weakest mandate (the spec neither includes nor explicitly
  defers them; only the RE report places them post-v1), so quotes do not start
  until the owner confirms the increment. The other four rest on the spec's
  own deferral rows or, for discrepancy, on ADR 0012's revisit trigger.
- The money model stays the single `paid` boolean through all five.
  `shortfall_total` on a discrepancy and `amount` on a payment link are
  captured, informational figures; real receivables, partial payments and
  per-reseller A/R remain a separate, later slice (functional spec §12).
- The backend keeps one persistence style (Panache over Flyway, ADR 0013) and
  one public hostname with the API under `/api` (ADR 0010); the only new
  outside dependency is Xendit for the last feature, with secrets outside the
  chart (ADR 0011) and the client stubbed so CI stays Docker-free (ADR 0005)
  and makes no external calls.
- Revisit the order if a pilot's needs invert the ranking — a pilot recording
  short deliveries before it needs differentiated pricing would pull
  discrepancy ahead of tiers, which the dependency graph allows (only
  quotes→tiers is a hard dependency).
