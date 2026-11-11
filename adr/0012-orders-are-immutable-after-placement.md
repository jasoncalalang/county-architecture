# ADR 0012: Orders are immutable after placement; a change is cancel and reorder

Date: 2026-10-31
Status: accepted

## Context

A reseller who typed 10 instead of 1 will want to fix it, and a wholesaler
who delivered 8 of 10 will want the order to say so. Letting orders change
after placement means an edit endpoint, recomputed totals, a rule for who
may edit in which status, and a dashboard that may have been read before
the edit.

Alternatives considered: the reseller edits lines while the order is
placed, which is the common web-shop pattern but means a wholesaler who has
seen an order may be looking at stale lines; the wholesaler edits any order
before delivery, which blurs what the reseller agreed to; both, with an
audit trail, which is a feature slice of its own.

## Decision

- An order's lines, note, delivery date and total never change after
  placement. Only its status and paid flag do.
- A reseller fixes a mistake by cancelling the placed order on their link
  and ordering again. The wholesaler can cancel a placed or confirmed order
  for the same reason.
- There is no edit endpoint. Short deliveries and other differences between
  what was ordered and what was delivered are a discrepancy report, deferred
  past v1.

## Consequences

- The status machine stays four states and five transitions, each with one
  test.
- The production run always reflects exactly the confirmed orders as
  placed; nothing moves under the kitchen.
- A corrected order has a new number. The old one stays visible as
  cancelled, so history adds up.
- A confirmed order cannot be corrected by the reseller; they phone the
  wholesaler, who cancels it, and they order again. Acceptable for next-day
  delivery volumes.
- Revisit together with discrepancy reports, when a pilot customer's
  short deliveries need recording.
