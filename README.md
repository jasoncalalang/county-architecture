# county architecture

Design documents and architecture decision records for **county**, an ordering
platform for Philippine SMEs. Version one lets a food wholesaler take orders
from independent resellers through a link sent to their mobile number, with
no login on the reseller side.

This repository holds the *why* and the *shape*. The code lives in the
`county` repository.

## Contents

- [`design/`](design/) — design documents, one per version or slice.
  - [2026-10-16 v1 design](design/2026-10-16-v1-design.md): wholesaler ordering for resellers. Approved.
  - [2026-10-31 v1 Kubernetes deployment](design/2026-10-31-v1-kubernetes-deployment.md): how v1 runs on the xeon cluster and how a commit becomes a release. Proposed.
  - [2026-10-31 v1 features and functionality](design/2026-10-31-v1-functional-spec.md): what each role can do, screen by screen, with acceptance criteria and the demo script. Proposed.
  - [2026-10-31 v1 interface design](design/2026-10-31-v1-ui-design.md): visual rules, components, every dashboard and order-page screen, states, responsive and accessibility rules. Proposed.
- [`adr/`](adr/) — architecture decision records, numbered, never edited after
  acceptance; a later ADR supersedes an earlier one.
- [`research/`](research/) — working notes that feed a design; superseded once the design records the outcome.
  - [2026-10-31 interface reference brief](research/2026-10-31-ui-reference-brief.md): comparables, Mobbin searches per screen, adoption rule, Superdesign order, and the findings of the Mobbin pass.
  - [0001](adr/0001-quarkus-api-nextjs-web.md) Quarkus API and Next.js web in one repository
  - [0002](adr/0002-google-sign-in-for-wholesalers.md) Google sign-in for wholesaler accounts
  - [0003](adr/0003-resellers-order-from-a-token-link.md) Resellers order from a per-reseller token link, no account
  - [0004](adr/0004-no-realtime-in-v1.md) No realtime transport in v1
  - [0005](adr/0005-tests-run-without-docker.md) The test suite must run without Docker
  - [0006](adr/0006-jenkins-deploys-with-helm.md) Jenkins deploys with Helm on every push to main
  - [0007](adr/0007-kaniko-images-tagged-by-commit.md) Images are built by Kaniko from Dockerfiles and tagged by commit SHA
  - [0008](adr/0008-postgres-statefulset-on-longhorn.md) Postgres runs in-cluster as one StatefulSet on Longhorn
  - [0009](adr/0009-flyway-migrates-at-api-start.md) Flyway migrates at API start; migrations stay compatible with the previous release
  - [0010](adr/0010-one-hostname-api-under-api-path.md) One public hostname, the API under `/api`, Auth.js under `/auth`
  - [0011](adr/0011-secrets-created-outside-the-chart.md) Secrets are created outside the chart and referenced by name
  - [0012](adr/0012-orders-are-immutable-after-placement.md) Orders are immutable after placement; a change is cancel and reorder

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
| v1: Kubernetes deployment | Design proposed, awaiting review |
| v1: features and functionality | Spec proposed, awaiting review |
| v1: interface design | Design proposed, awaiting review |
