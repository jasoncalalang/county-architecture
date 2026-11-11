# ADR 0011: Secrets are created outside the chart and referenced by name

Date: 2026-10-31
Status: accepted

## Context

v1 has three secrets per environment: the Google OAuth client id and secret,
the Postgres password and the Auth.js signing secret. They have to reach the
pods without entering git, Helm values, Jenkins logs or Helm's release
history, which stores every value passed to a release.

Alternatives considered: Secrets templated from values passed with `--set`,
which lands the values in Helm release Secrets and in shell history; Sealed
Secrets, which encrypts them for git and needs a controller and a key to
back up; External Secrets Operator, which needs a vault we do not run.

## Decision

- The operator creates `county-google`, `county-postgres` and `county-web`
  once per namespace with `kubectl create secret`. Keys are UPPER_SNAKE and
  match the environment variables they feed.
- The chart names them in values and references them with `secretKeyRef`.
  It never renders a `Secret` resource.
- Rotation is recreating the Secret and `kubectl rollout restart` on the
  Deployments that read it.

## Consequences

- The first install has a documented manual step; every later release has
  none.
- The chart is reproducible from git plus three Secrets.
- A missing Secret fails the release with a message naming it, and
  `--atomic` rolls back.
- Revisit with Sealed Secrets when more than one person rotates secrets, or
  with External Secrets if a vault appears.
