# county architecture

Design documents and architecture decision records for **county**, an ordering
platform for Philippine SMEs. Version one lets a food wholesaler take orders
from independent resellers through a link sent to their mobile number, with
no login on the reseller side.

This repository holds the *why* and the *shape*. The code lives in the
`county` repository.

## Contents

- [`design/`](design/) — approved design documents, one per version or slice.
  - [2026-10-16 v1 design](design/2026-10-16-v1-design.md): wholesaler ordering for resellers.
- [`adr/`](adr/) — architecture decision records, numbered, never edited after
  acceptance; a later ADR supersedes an earlier one.

## Principles

1. **Ship the smallest loop that a customer would pay for**, then grow. v1 is
   invite reseller → reseller orders → confirm → production run → deliver.
2. **The tenant comes from the credential, never from the request.** No
   endpoint accepts a tenant id. Cross-tenant access is a 404.
3. **Rules live on the server.** Delivery dates, totals and status transitions
   are computed and enforced by the API; the web client only displays them.
4. **The reseller side must work on a cheap phone inside a chat app's
   browser.** Server-rendered, minimal JavaScript, stateless links.
5. **Tests first.** Every rule has a failing test before it has code, and the
   test suite must run on a machine without Docker.
6. **Boring errors.** HTTP status codes and Problem Details, nothing custom.

## Status

| Version | Status |
|---|---|
| v1: wholesaler ordering | Design approved, implementation planning |
