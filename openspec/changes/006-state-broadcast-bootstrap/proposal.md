# Bootstrap state-broadcast capability

## Why

The Event Stream Server module (MOD-0002) is the bridge between the
simulator's state changes and every connected browser. Capability
`sse-connection-management` (CAP-0003, proposal 003) handles the
connection lifecycle — accept, track, clean disconnect. Capability
`state-broadcast` (CAP-0004) is the matching half: when the simulator
emits a state-change notification (per CAP-0002), the broadcast
serialises it as a text/event-stream event and pushes it across the
registry of currently-connected clients.

Without this spec the 5-second latency ceiling from Ch 2 (a hard
functional constraint of the PoC) has no per-event observable
anchor, the async-at-most-once consistency posture on the producer-
side flow ENT-0001: MOD-0002 → MOD-0003 has no behaviour-level
encoding, and the SSE event payload shape on the wire — what the
browser must parse in CAP-0006 — is ambiguous.

## What

Adds one OpenSpec spec at `specs/state-broadcast/spec.md` with four
requirements covering: receiving each state-change notification from
the simulator and broadcasting it across the connection registry that
capability `sse-connection-management` maintains; serialising each
notification as a text/event-stream event whose data field carries the
six flow-defined fields; the p95 < 5-second end-to-end latency budget
on the broadcast path (anchored to the Ch 2 hard constraint and the
IFC-0001 / IFC-0002 SLO); and the async-at-most-once posture (no per-
client retry on send failure; the broadcast may drop a notification
on back-pressure rather than block the simulator). Every requirement
is ADDED.

## Out of scope

- Acceptance and lifecycle management of the connections themselves
  (CAP-0003 territory; covered in proposal 003).
- Production of the state-change notifications — the simulator's
  emission of a notification per state change (CAP-0002 territory;
  covered in proposal 004).
- Browser-side parsing and state application of received events
  (CAP-0006 territory; covered in proposal 005).
- Any per-client filtering — every connected browser receives every
  flight's events (the dashboard shows all active flights to every
  viewer; no scoping interface exists).

## Anchors

- Ch 4 CAP-0004 (target capability)
- Ch 4 MOD-0002 (parent module — Event Stream Server)
- Ch 4 § Intermodule Flows: ENT-0001 MOD-0001 → MOD-0002 (consumer side)
- Ch 4 § Intermodule Flows: ENT-0001 MOD-0002 → MOD-0003 via IFC-0001 (producer side)
- FND-0002 (Business exception handling — passive degradation)
- FND-0003 (Async & event architecture — SSE / async-at-most-once)
- FND-0004 (Observability — structured logging)
- IFC-0001 (text/event-stream interface to ACT-0001 — p95 < 5000ms)
- IFC-0002 (text/event-stream interface to ACT-0002 — p95 < 5000ms)
- Ch 2 § Constraint — "Display latency ceiling of 5 seconds"
