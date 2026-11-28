# county-architecture

Design documents and ADRs for county. The code and its engineering rules
(TDD, SOLID, YAGNI, security by design, the plan-build-review loop) are in
the `county` repository's `CLAUDE.md`.

- ADRs are never edited after acceptance; a later ADR supersedes one.
- A design document or build plan has a **Security** section: the assets,
  actors (owner, reseller holding a link, anonymous caller, owner of another
  tenant), entry points and abuse cases, each with its control. Plans in
  `county` turn each abuse case into a test.
- A change that weakens tenant isolation, authentication, validation or
  secrets handling needs its own ADR.
- Record decisions only Jason has made. Mark anything else as proposed.
- Rate limiting stays at the Cloudflare edge.
