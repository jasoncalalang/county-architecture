# ADR 0002: Google sign-in for wholesaler accounts

Date: 2026-10-16
Status: accepted

## Context

Wholesaler users need to log in. Running an identity server (Keycloak) is the
team's usual pattern but adds a component to operate for a demo-stage product.
Nearly every Philippine SME owner already has a Google account.

## Decision

- The web app uses Auth.js with the Google provider for the wholesaler session.
- The API verifies Google ID tokens as bearer tokens through `quarkus-oidc`
  with `quarkus.oidc.provider=google`. Nothing identity-related is hosted by us.
- Identities are stored as (provider, subject) in their own table, mapped to a
  `user` row. Adding another provider later is configuration and one row per
  identity, not a schema change.
- Resellers do not log in at all; see ADR 0003.

## Consequences

- No identity server to deploy or back up.
- Wholesalers without a Google account cannot sign up in v1.
- If Google is unavailable, wholesalers cannot sign in; accepted for v1.
