# Bootstrap event-simulation capability

## Why

Capability `flight-generation` (CAP-0001) populates the active flight
pool but does not move passengers through check-in. The dashboard's
entire reason for existing — observing live progress across flights —
depends on a steady stream of state changes that progress passengers
from "not checked in" to "checked in (online or counter)", assign them
seats, and progress their baggage through registration and drop-off.
That progression is what `event-simulation` (CAP-0002) produces.

This proposal bootstraps the spec for that progression: which entity
state changes the simulator generates, how each change rolls up into
the Flight aggregate counts the dashboard displays, and the cross-module
notification to the Event Stream Server (the flow ENT-0001:
MOD-0001 → MOD-0002, async-at-most-once, fields {flightId,
checkedInOnline, checkedInCounter, bagsRegistered, bagsDropped,
seatsAssigned} per Ch 4 § Intermodule Flows). Without this spec the
dashboard would render whatever bootstrap counts CAP-0001 set and
never change.

## What

Adds one OpenSpec spec at `specs/event-simulation/spec.md` with five
requirements covering: passenger check-in events progressing each
PassengerRecord with a channel and check-in time, with the matching
Flight aggregate count incrementing; seat-assignment events updating
PassengerRecord and Flight.seatsAssigned; baggage state transitions
through registered → dropped, with the matching Flight aggregates
incrementing; the cross-module state-change notification published on
each event per the flow contract; and the steady, uniform-rate
generation posture from Ch 2 (no peak-load bursts). Every requirement
is ADDED.

## Out of scope

- Bootstrap and eviction of flights / passengers / baggage records
  (CAP-0001 territory; covered in proposal 002).
- Delivery of the state-change notification across browser SSE
  connections (CAP-0004 territory; deferred to its own proposal).
- Any reverse path from the dashboard back to the simulator — there is
  no such path; the application is read-only (Ch 2 explicit non-goal).
- Specific arrival-rate parameter values (an L3 simulator parameter,
  not an L2 contract; Ch 2 fixes only "steady, uniform").

## Anchors

- Ch 4 CAP-0002 (target capability)
- Ch 4 MOD-0001 (parent module — Flight Data Simulator)
- Ch 4 § Intermodule Flows: ENT-0001 MOD-0001 → MOD-0002
- FND-0002 (Business exception handling — passive degradation)
- FND-0005 (Application Platform — Rust + Tokio)
- ENT-0001 (Flight — aggregate counts updated)
- ENT-0002 (PassengerRecord — channel, check-in time, seat updated)
- ENT-0003 (BaggageRecord — status transitions)
- Ch 2 § Constraint — "steady, uniform arrival of events (no peak-load bursts)"
