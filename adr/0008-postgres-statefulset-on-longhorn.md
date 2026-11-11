# ADR 0008: Postgres runs in-cluster as one StatefulSet on Longhorn

Date: 2026-10-31
Status: accepted

## Context

The v1 design puts Postgres on Longhorn storage. The question is what runs
it: a StatefulSet in our own chart, a third-party Postgres chart, an operator
such as CloudNativePG, or a database outside the cluster. v1 is a demo on a
single node, but demos are rehearsed with data someone typed in, so a
restore path still matters.

Alternatives considered: CloudNativePG, which gives streaming replicas,
backups to object storage and point-in-time recovery, and is the right
answer for a pilot, but is an operator to install and learn now; a
third-party Postgres chart, which brings dozens of values and someone else's
image policy for a one-pod database; a managed database, which xeon has no
provider for and which puts the database on the far side of the tunnel.

## Decision

- Postgres is a StatefulSet template in the county chart: the official
  `postgres` image, one replica, one 10Gi Longhorn PVC from
  `volumeClaimTemplates`, `pg_isready` probes.
- Backups are Longhorn recurring snapshots of the volume, daily, retain 7.
  Restore is a volume revert with api scaled to zero, rehearsed before the
  first demo.
- `postgres.enabled=false` plus an external host points api at a database
  outside the chart, so moving to CloudNativePG or a managed service does
  not change api or web.

## Consequences

- A Postgres restart takes the API down for about thirty seconds; readiness
  fails and the Ingress returns 503. Accepted.
- No point-in-time recovery: a lost disk loses up to a day. Accepted for a
  demo, not for a customer, which is the trigger to move to CloudNativePG.
- Major version upgrades are a manual dump and restore outside the chart.
- The chart owns the database lifecycle, so `helm uninstall` must not delete
  the PVC; the StatefulSet's retention policy keeps it.
