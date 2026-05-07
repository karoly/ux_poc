# Bootstrap sse-connection-management capability

## Why

The Event Stream Server module (MOD-0002) is the single bridge between
the simulator's in-memory state changes and every browser viewing the
dashboard. Two capabilities sit inside MOD-0002: `sse-connection-management`
(CAP-0003) handles the lifecycle of each browser connection — accept,
track, clean disconnect; `state-broadcast` (CAP-0004) pushes events
across the registry of active connections. This proposal bootstraps
the first capability; the second follows in its own proposal.

Without an explicit spec for connection lifecycle, three of the
foundation decisions affecting MOD-0002 lack an implementation anchor:
the SSE protocol selection in FND-0003, the passive-degradation posture
on stream interruption in FND-0002 ("On client disconnect the
connection is closed cleanly; no event replay is provided on
reconnection"), and the structured-logging observability posture in
FND-0004. The "no event replay on reconnection" claim in particular is
behaviour-defining for the PoC's evaluation context — without it,
implementation may drift toward backlog buffering, which adds
complexity that does not contribute to the usability evaluation.

## What

Adds one OpenSpec spec at `specs/sse-connection-management/spec.md`
with four requirements covering: acceptance of persistent SSE client
connections with the correct response shape; maintenance of an
in-memory registry of currently-connected clients (so capability
`state-broadcast` can iterate over it); clean handling of client
disconnects with registry cleanup; and the explicit "no replay, no
backlog" posture for newly-arriving (or reconnecting) clients. Every
requirement is ADDED.

## Out of scope

- Pushing flight state-change events across connected clients
  (CAP-0004 territory; deferred to its own proposal).
- Authentication or authorization on the SSE endpoint (FND-0006 § N/A
  / Ch 2 explicit non-goal — auth is not in scope for the PoC).
- TLS termination on the SSE endpoint (Ch 5 § security_perimeter — all
  ports are loopback-bound; no TLS required).
- Latency budgets on individual events (CAP-0004 / FND-0003's SLO
  applies to delivery, not connection establishment).

## Anchors

- Ch 4 CAP-0003 (target capability)
- Ch 4 MOD-0002 (parent module — Event Stream Server)
- FND-0002 (Business exception handling — passive degradation on disconnect)
- FND-0003 (Async & event architecture — SSE protocol selection)
- FND-0004 (Observability — structured logging of connection lifecycle)
- FND-0005 (Application Platform — Rust + Tokio async I/O)
- IFC-0001 (server interface to ACT-0001 — text/event-stream)
- IFC-0002 (server interface to ACT-0002 — text/event-stream)
- ACT-0001 (ground-handling staff — connecting via browser)
- ACT-0002 (Amadeus Technical Leadership — connecting via browser)
