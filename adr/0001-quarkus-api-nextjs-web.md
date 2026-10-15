# ADR 0001: Quarkus API and Next.js web in one repository

Date: 2026-10-16
Status: accepted

## Context

county needs a wholesaler dashboard, a public reseller order page that must
work in chat-app browsers on low-end phones, and an API that enforces tenant
isolation and ordering rules. The team's deepest backend skill is Java on
Quarkus; the frontend skill is TypeScript on Next.js. The product runs on a
self-managed Kubernetes cluster with in-cluster CI.

Alternatives considered: a single Next.js application with server actions and
a TypeScript ORM (fastest to demo, but the domain logic would be bound to the
web framework); Flutter on a hosted backend (good for a wholesaler mobile app,
but Flutter Web is too heavy for the reseller page).

## Decision

- `api/`: Quarkus (Java 21), RESTEasy Reactive, Hibernate ORM with Panache,
  Flyway, Postgres, SmallRye OpenAPI.
- `web/`: Next.js (App Router, TypeScript). The reseller page is server-rendered.
- Both live in the `county` repository with one Helm chart. Splitting into
  separate repositories is deferred until the API contract stabilises.
- The web API client is generated from the API's OpenAPI document at build
  time, so contract drift fails compilation.

## Consequences

- Two deployables and an API contract from day one, which costs some demo
  speed but keeps domain rules in a place built for them.
- Java and TypeScript toolchains both required in CI.
- The reseller page carries almost no client JavaScript, which constrains UI
  choices on that page to forms and links.
