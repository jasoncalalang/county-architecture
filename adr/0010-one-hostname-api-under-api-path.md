# ADR 0010: One public hostname, the API under /api, Auth.js under /auth

Date: 2026-10-31
Status: accepted

## Context

The wholesaler dashboard is client-rendered and calls the API from the
browser with a Google ID token, so the API must be reachable from the
internet. The reseller page is server-rendered and calls the API from the
web pod. Both deployables sit behind one Cloudflare tunnel.

Alternatives considered: a second hostname such as `api.county.<domain>`,
which needs a second tunnel route and DNS record, CORS configuration on the
API and a preflight on every mutating call; Next.js rewrites that proxy
`/api` to Quarkus, which keeps one hostname but puts Node in the path of
every API call and makes web's availability a precondition of the API's.

## Decision

- One hostname. The Ingress routes the prefix `/api` to `county-api` and
  everything else to `county-web`.
- Quarkus serves under `quarkus.http.root-path=/api`. Its non-application
  endpoints (health, metrics, OpenAPI) use the absolute path `/q`, which the
  Ingress does not route.
- Auth.js is mounted at `/auth` through `basePath`, not its default
  `/api/auth`, so nothing of web lives under `/api`. The Google redirect URI
  is `https://<host>/auth/callback/google`.
- The browser calls the API at the relative URL `/api`. The web server calls
  `http://county-api:8080/api`. Same path, two base URLs.
- Both apps are configured as proxied: Quarkus forwards proxy headers,
  Auth.js trusts the host and has `AUTH_URL` set.

## Consequences

- No CORS, one DNS record, one tunnel route, one redirect URI per
  environment.
- The web image carries no environment-specific URL, so one image serves
  every environment.
- The Ingress is the only place that knows about both services. Moving the
  API to its own hostname later is a change to the Ingress and to web's base
  URL, not to either application.
- The non-default Auth.js path must be remembered in the Google console and
  in every web route that links to sign-in.
