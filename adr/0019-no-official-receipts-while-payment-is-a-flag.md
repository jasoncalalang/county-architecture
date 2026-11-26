# ADR 0019: No official receipts while payment is a flag

Date: 2026-11-26
Status: accepted

## Context

Philippine businesses issue BIR-registered official receipts or invoices,
with a TIN, a registered serial series and VAT treatment. Itemcount
implements X and Z readings, an e-journal and a BIR sales report. county
records money only as a `paid` flag
([functional spec](../design/2026-10-31-v1-functional-spec.md) §2, §12),
and the v1 design lists POS and BIR receipts as out of scope. Payment links
(ADR 0014) record payments but still not receipts.

## Decision

- county issues no official receipts or invoices and stores no TIN, receipt
  series or VAT-exempt flag while money is recorded as the `paid` flag and
  payment-link events.
- No screen, export or message may call a county order or payment record a
  receipt or invoice.
- Revisit before county records receivables, partial payments or anything
  a business would file with the BIR.

## Consequences

- Wholesalers keep issuing receipts from their own system or booklet.
- When this is revisited, the schema work (TIN, series, VAT flag) and the
  compliance work need their own design and ADR.
