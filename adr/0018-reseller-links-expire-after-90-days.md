# ADR 0018: Reseller order links expire after 90 days

Date: 2026-11-26
Status: accepted

## Context

[ADR 0003](0003-resellers-order-from-a-token-link.md) makes the token in a
reseller's link the only credential, and its mitigations are long random
tokens, block and reissue, and the wholesaler seeing every order. Tokens
never expire, so a link forwarded once works forever. The public order
endpoint is also the only unauthenticated write (county playbook W2).

Alternatives considered: no expiry (the status quo); a short expiry such as
7 days (resellers order weekly or less and would hit dead links); expiry
reset on every order (a link in steady use never expires, so a leaked one
never does either).

## Decision

- A reseller's token expires 90 days after it is issued.
- Reissuing a link gives a new token and a new 90 days; the old token stops
  working at once, as today.
- An expired token behaves exactly like a blocked one: the same 404 and the
  same "This link is no longer active" page. There is no new error shape.
- Existing tokens are backfilled to expire 90 days after the release that
  adds expiry.

## Consequences

- Wholesalers must reissue links every 90 days. The dashboard should show
  when each link expires; that is a web follow-up.
- ADR 0003 stands; this adds a lifetime to its token.
- Rate limiting of the public order endpoint (W2) is separate and needs no
  ADR.
