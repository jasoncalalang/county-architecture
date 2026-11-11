# ADR 0006: Jenkins deploys with Helm on every push to main

Date: 2026-10-31
Status: accepted

## Context

The v1 design has the in-cluster Jenkins build images and the xeon cluster
run them. Something has to apply a release to the cluster: the pipeline
itself (push), a GitOps controller watching a manifest repository (pull), or
a person running Helm. v1 has one environment, one deployer and a demo
audience.

Alternatives considered: Argo CD or Flux, which reconcile the cluster to a
git state and show drift, but need a manifest or chart repository to watch
and one more component to install and operate; a person running
`helm upgrade` after each merge, which puts a human in every release.

## Decision

- On `main`, after tests, images and `helm lint` pass, Jenkins runs
  `helm upgrade --install county deploy/county --atomic --wait --timeout 5m`
  with the image tag set to the commit SHA.
- The chart is applied from the working tree of the built commit. No chart
  repository, no chart publishing.
- Other branches build and test, including images, but never deploy.
- Rolling back is `helm rollback` by hand or a revert on `main`.

## Consequences

- Cluster state is whatever the last successful `main` build applied. A
  change made with `kubectl edit` is overwritten by the next release, which
  is the intent.
- `--atomic` means a release that does not become ready in five minutes is
  undone without anyone watching.
- Jenkins holds the only deploy credential, scoped to the `county-demo`
  namespace. That namespace is the blast radius of a compromised Jenkins.
- Revisit when a second environment or a second deployer appears; GitOps is
  the expected successor and the chart is already shaped for it.
