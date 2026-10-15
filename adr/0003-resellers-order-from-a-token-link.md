# ADR 0003: Resellers order from a per-reseller token link, no account

Date: 2026-10-16
Status: accepted

## Context

The wholesaler's buyers are independent resellers: market stalls, sari-sari
stores, carinderias. They will not install an app, manage a password or use a
dashboard. They do use Viber, Messenger and SMS on inexpensive Android phones.

## Decision

- Each reseller has a random, unguessable order token. The wholesaler sends
  the link `…/order/{token}` by any channel; in v1 this is a manual copy and
  paste.
- The order page is stateless: the token in the URL is the only credential.
  No cookie, no password, no session.
- The API returns 404 for unknown and blocked tokens alike, so tokens cannot
  be probed and blocking a reseller is immediate.
- Order submission carries an idempotency key per form render; a repeated
  submission returns the original order.

## Consequences

- Anyone holding the link can order as that reseller. Mitigations: tokens are
  long and random, the wholesaler can block and reissue, and every order is
  visible to the wholesaler before it is confirmed.
- No reseller-side history beyond what the link page shows.
- An SMS gateway for sending links is a later feature.
