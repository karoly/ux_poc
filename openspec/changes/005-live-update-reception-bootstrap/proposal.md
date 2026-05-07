# Bootstrap live-update-reception capability

## Why

The Check-in Monitor Dashboard module (MOD-0003) splits into two
capabilities: `dashboard-layout` (CAP-0005) renders the per-flight
status grid and `live-update-reception` (CAP-0006) connects to the
Event Stream Server, receives server-pushed events, and applies them
to the client-side state the layout renders. CAP-0005's spec
(proposal 001) refers throughout to "the client-side state"; this
proposal anchors what produces that state and how it stays current.

Without an explicit spec, three of the foundation decisions affecting
MOD-0003 lack an implementation anchor: FND-0003's SSE protocol
selection on the consumer side (the browser EventSource), FND-0002's
passive-degradation posture on stream interruption (the display
freezes at the last-received state, no error indicator), and the
async-at-most-once consistency posture on the Ch 4 flow ENT-0001:
MOD-0002 → MOD-0003 (IFC-0001 / IFC-0002 carry the flow-defined
field set without retry).

## What

Adds one OpenSpec spec at `specs/live-update-reception/spec.md` with
four requirements covering: opening the SSE connection to the Event
Stream Server when the dashboard mounts; parsing each incoming event
into the flow-defined field set (flightId, checkedInOnline,
checkedInCounter, bagsRegistered, bagsDropped, seatsAssigned); applying
each event to the client-side flight state by flightId so the rendered
grid reflects the latest values without a page reload; and the
passive-degradation behaviour on stream interruption from FND-0002.
Every requirement is ADDED.

## Out of scope

- Rendering of the grid and per-flight cards (CAP-0005 territory;
  covered in proposal 001).
- Server-side acceptance and lifecycle management of the connection
  (CAP-0003 territory; covered in proposal 003).
- Server-side broadcasting of events to all connected clients (CAP-0004
  territory; covered separately).
- Any client-to-server message — there is none; the stream is
  unidirectional per FND-0003.
- Reconciliation of "missed" events on reconnection — none is
  performed; FND-0002 / CAP-0003's no-replay posture means a
  reconnecting client receives only events arriving after reconnection.

## Anchors

- Ch 4 CAP-0006 (target capability)
- Ch 4 MOD-0003 (parent module — Check-in Monitor Dashboard)
- Ch 4 § Intermodule Flows: ENT-0001 MOD-0002 → MOD-0003 via IFC-0001
- FND-0002 (Business exception handling — passive degradation)
- FND-0003 (Async & event architecture — SSE / EventSource)
- FND-0006 (Front-end Framework — React)
- IFC-0001 (text/event-stream interface served to ACT-0001)
- IFC-0002 (text/event-stream interface served to ACT-0002)
- ACT-0001, ACT-0002 (browsers operated by these actors host this capability)
