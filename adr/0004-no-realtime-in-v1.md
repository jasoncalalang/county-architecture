# ADR 0004: No realtime transport in v1

Date: 2026-10-16
Status: accepted

## Context

A wholesaler dashboard benefits from seeing new orders as they arrive. Push
transports (WebSocket, server-sent events) add a long-lived connection
component, reconnection logic and a second path for every data change.

## Decision

The dashboard polls the orders endpoint every 30 seconds. No WebSocket or
SSE in v1.

## Consequences

- Up to 30 seconds of delay before a new order shows; acceptable for orders
  that are produced and delivered the next day.
- One request path for every piece of data, which keeps tenant isolation and
  testing simple.
- If a later slice needs push (for example, a counter display), it will be
  added as a cache-invalidation signal, not a second data channel.
