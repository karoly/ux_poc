# Bootstrap flight-generation capability

## Why

The Flight Data Simulator module (MOD-0001) is the sole data source for
the entire PoC: every count the dashboard displays originates from
flight, passenger, and baggage records this module owns. Two capabilities
sit inside MOD-0001: `flight-generation` (CAP-0001) bootstraps and
maintains the pool of active flights; `event-simulation` (CAP-0002)
produces the continuous check-in / baggage events on top of that pool.

This proposal bootstraps the first OpenSpec spec for `flight-generation`
so the contract for the active flight pool — its minimum size, the
check-in window definition, and the eviction policy at departure — is
captured before any backend work begins. Without this spec, the
guarantee that "at least 100 simultaneously active flights" is in
scope (a Ch 2 hard constraint) and the data-lifecycle decision in
FND-0001 ("evict on departure") have no implementation anchor.

## What

Adds one OpenSpec spec at `specs/flight-generation/spec.md` with four
requirements covering: initial pool bootstrap at process startup; the
24-hour check-in window posture from Ch 2; steady-state pool
maintenance (continuous introduction of new flights as old ones
depart); and eviction at departure time with cascade through the
passenger and baggage records owned by the evicted flight. Every
requirement is ADDED — there is no prior `flight-generation` spec.

## Out of scope

- Continuous passenger check-in, seat-assignment, and baggage events
  (CAP-0002 territory; deferred to its own bootstrap proposal).
- Outbound delivery of state changes to the Event Stream Server
  (`state-broadcast` CAP-0004 territory).
- Specific arrival-rate distribution for new flights (an L3 simulator
  parameter, not an L2 contract; Ch 2 fixes only "steady, uniform").
- Persistence of any state (FND-0001 explicitly forbids a durable
  store; in-memory only).

## Anchors

- Ch 4 CAP-0001 (target capability)
- Ch 4 MOD-0001 (parent module — Flight Data Simulator)
- FND-0001 (Data lifecycle & retention — evict on departure)
- FND-0005 (Application Platform — Rust + Tokio in-process)
- ENT-0001 (Flight — owned by MOD-0001)
- ENT-0002 (PassengerRecord — owned by MOD-0001, child of Flight)
- ENT-0003 (BaggageRecord — owned by MOD-0001, child of PassengerRecord)
- Ch 2 § Constraint — "at least 100 simultaneously active flights"
- Ch 2 § What the product does — "check-in window opens 24 hours before departure"
