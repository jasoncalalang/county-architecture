# ADR 0017: Fractional quantities round half up per line

Date: 2026-11-26
Status: accepted

## Context

Functional spec §5.3 makes quantity a whole number from 1 to 9999: "Half a
kilo is a separate product". Wholesalers of rice, meat and produce take
orders such as 1.5 kg, so the county playbook (W1) widens quantity to three
decimals. A price in centavos times a three-decimal quantity can produce
fractions of a centavo (₱99.99 × 0.333 = ₱33.29667), so a rounding rule
is needed. `Money.line` refuses to
round today.

Alternatives considered: round only the order total (line totals would not
add up to what is shown); banker's rounding (unfamiliar to resellers and
the wholesaler checking a receipt by hand); truncation (always favours the
reseller, which is a hidden discount).

## Decision

- Quantity is a decimal greater than 0 and at most 9999, with up to three
  decimal places. A fourth decimal is a 422.
- A line total is unit price × quantity, rounded half up to centavos
  (`RoundingMode.HALF_UP`, scale 2).
- The order total is the sum of the rounded line totals, never the rounded
  sum of unrounded lines.
- The captured `order_line` price and total stay `numeric(12,2)` and keep
  ADR 0012's immutability.

## Consequences

- Functional spec §5.3's "No decimals in v1" is superseded once W1 lands;
  the spec is updated then.
- `Money.line` gains the rule, with tests at ₱120.00 × 0.333 = ₱39.96 (exact)
  and ₱99.99 × 0.333 = ₱33.30 (rounds up).
- The production run sums decimal quantities.
