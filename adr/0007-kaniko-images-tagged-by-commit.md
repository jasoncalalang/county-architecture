# ADR 0007: Images are built by Kaniko from Dockerfiles and tagged by commit SHA

Date: 2026-10-31
Status: accepted

## Context

Jenkins runs on xeon, which has containerd and no Docker daemon (ADR 0005).
Images for `api` and `web` must be built inside an unprivileged pod and
pushed to GHCR. The two deployables have different toolchains, Gradle and
npm.

Alternatives considered: Jib for `api` (no Dockerfile, no daemon, one Gradle
flag) but `web` would still need another tool; Buildah, which needs
fuse-overlayfs or a privileged pod; Docker-in-Docker, which needs the
privilege we chose not to have; building on a developer machine, which is
not reproducible.

Kaniko's original Google repository was archived in June 2025. Maintained
forks exist (Chainguard's, limited to security and dependency updates, and a
community fork that keeps adding features). The tool still does exactly what
we need with no privileges.

## Decision

- Both images are described by a Dockerfile, `api/Dockerfile` and
  `web/Dockerfile`, and built by a Kaniko executor container in the Jenkins
  pod with the repository root as context.
- The executor image is a maintained fork's, pinned by tag, and recorded in
  the `Jenkinsfile`. Replacing it with rootless BuildKit later changes one
  pipeline stage and no Dockerfile.
- `api` is JVM mode on the Quarkus `Dockerfile.jvm` base. Native compilation
  is deferred.
- `web` is Next.js standalone output on a current Node LTS Alpine base, run
  as a non-root user.
- Tags are the full 40-character commit SHA. No `latest`, no floating tags,
  no tag pushed twice. Layer caches live in `<image>-cache` repositories.

## Consequences

- One build mechanism, and a Dockerfile is the whole description of an
  image; a developer can build the same thing locally with podman or
  nerdctl.
- Kaniko is slower than a daemon-backed build; with layer caching the
  difference is a minute or two per image.
- A JVM `api` image needs roughly 400Mi at rest where native would need a
  tenth of that. Accepted for a demo on a node we own.
- Every running pod can be traced to a commit from its image tag alone.
