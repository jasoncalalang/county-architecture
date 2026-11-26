# ADR 0016: A reseller's order is a sales order

Date: 2026-11-26
Status: accepted

## Context

The order a reseller places with the wholesaler is stored as
`purchase_order`. In accounting, and in Itemcount, a purchase order is what a
business buys from its supplier; what it sells to a customer is a sales
order. county will add stock and, later, sourcing from suppliers
(functional spec §12). At that point `purchase_order` would name the
opposite of what it holds. There is no production data yet, so a rename
costs one migration and a code change.

Alternatives considered: keep the name and call the supplier side something
else (keeps a misleading name for good); rename later (costs more once a
release with data exists, under ADR 0009's expand-first rule).

## Decision

- The table `purchase_order` becomes `sales_order`, with its constraints and
  indexes renamed to match, in one forward-only Flyway migration with no
  data change.
- The Java entity and service names follow in the same pull request, so the
  table and the code never disagree.
- The API paths (`/orders`, `/order/{token}/orders`) and the JSON shapes do
  not change.
- "Purchase order" is reserved for orders the business places with its
  suppliers.

## Consequences

- Earlier ADRs and design documents keep saying `purchase_order`; read it as
  `sales_order`. They are not edited.
- The rename touches the placement code that price tiers just changed, so
  it lands as its own pull request (county playbook W4) with the existing
  suite unchanged as its test.
