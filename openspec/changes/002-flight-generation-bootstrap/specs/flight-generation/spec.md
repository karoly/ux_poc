# flight-generation: Active flight pool bootstrap and maintenance

## Purpose

This capability defines the active flight pool that all other behaviours
in the system read from: it bootstraps the initial pool at process
startup, continuously introduces new flights as their check-in windows
open, and removes flights from the in-memory store once their departure
time passes. It owns the lifecycle of three entities — Flight,
PassengerRecord, and BaggageRecord — which together represent the
simulated reality the dashboard displays. The continuous generation of
check-in, seat-assignment, and baggage events on top of this pool is
governed by capability `event-simulation` and is out of scope here.

## Dependencies

- Foundation decisions: FND-0001, FND-0005
- Modules: MOD-0001
- Entities: ENT-0001, ENT-0002, ENT-0003

## Requirements

### Requirement: Initial pool is bootstrapped with at least 100 active flights

<!-- forge-id: REQ-0005 -->

At process startup, the simulator SHALL populate the in-memory store
with at least 100 Flight records whose check-in windows are currently
open (departureTime is in the future and checkInWindowOpens is in the
past, by the simulator's clock). Each bootstrapped Flight SHALL be
created together with its full set of PassengerRecord rows (totalling
the flight's totalPassengers attribute) and each PassengerRecord SHALL
be created together with its associated BaggageRecord rows. Flights
created at bootstrap MAY be at any stage of check-in progress (some
passengers already checked in, others not) so that the dashboard does
not start in an artificial all-zero state. This requirement encodes the
Ch 2 hard constraint "at least 100 simultaneously active flights" and
the FND-0001 § Affects relationship.

#### Scenario: At least 100 flights are present after startup completes

- GIVEN the simulator process has just completed its startup phase
- WHEN the in-memory store is queried for flights with departureTime in
  the future and checkInWindowOpens in the past
- THEN the count returned is at least 100

#### Scenario: Each bootstrapped flight has its full passenger and baggage records

- GIVEN the simulator has completed startup and a bootstrapped Flight F
  has totalPassengers = 250
- WHEN the in-memory store is queried for PassengerRecord rows whose
  parent flight is F
- THEN exactly 250 rows are returned, and every one of those rows has
  the BaggageRecord rows associated with it that the simulator's
  parameters dictate (zero or more per passenger, depending on the
  simulator's bag-per-passenger model)

### Requirement: A flight's check-in window opens 24 hours before its departure time

<!-- forge-id: REQ-0006 -->

For every Flight in the in-memory store, the value of
checkInWindowOpens MUST equal the value of departureTime minus exactly
24 hours. The simulator SHALL NOT introduce a Flight into the active
pool whose checkInWindowOpens is later than departureTime minus 24
hours, nor earlier than departureTime minus 24 hours. This requirement
encodes the Ch 2 statement "The check-in window for any given flight
opens 24 hours before departure" and pins it as a structural invariant
of every Flight record so downstream capabilities can rely on the
relationship without recomputing it.

#### Scenario: Every flight in the store satisfies the 24-hour invariant

- GIVEN the simulator has been running and the in-memory store contains
  flights F1 through Fn
- WHEN the test harness inspects each Flight's checkInWindowOpens and
  departureTime
- THEN for every flight Fi, departureTime(Fi) - checkInWindowOpens(Fi)
  is exactly 24 hours

#### Scenario: A newly introduced flight satisfies the invariant at insertion

- GIVEN the simulator has decided to introduce a new flight whose
  departureTime is T
- WHEN the new Flight record is inserted into the in-memory store
- THEN the inserted Flight's checkInWindowOpens equals T minus 24 hours

### Requirement: The active pool is continuously refilled as flights depart

<!-- forge-id: REQ-0007 -->

Over the lifetime of the simulator process, the count of Flight records
in the in-memory store with departureTime in the future SHALL remain at
or above 100 at all times after startup completes. As flights are
removed from the store at departure (per REQ-0008), the simulator SHALL
introduce new Flight records (each accompanied by its passenger and
baggage records, per REQ-0005's bootstrap shape) at a rate that keeps
the active pool count from dropping below 100. The introduction rate
SHALL be steady and uniform — there are no peak-load bursts (Ch 2 §
Constraint).

#### Scenario: Pool count stays at or above 100 across simulated departures

- GIVEN the simulator has completed startup and a test harness sampling
  the active flight count every wall-clock second for one full simulator
  hour
- WHEN at least one bootstrapped flight has reached its departureTime
  during the sampling window and been evicted (per REQ-0008)
- THEN every sample taken during the window returns an active flight
  count of at least 100

#### Scenario: New flights enter the pool at a uniform rate, not in bursts

- GIVEN the simulator has been running long enough that bootstrapped
  flights have begun to reach departureTime
- WHEN the test harness records the wall-clock timestamp of each new
  Flight insertion over a one-hour window
- THEN no two inserts in the window are bunched into a burst pattern
  inconsistent with the simulator's stated steady, uniform arrival rate
  (the harness asserts the inter-arrival intervals fall within a
  bounded variance the simulator parameters specify)

### Requirement: Flights and their dependent records are evicted at departure time

<!-- forge-id: REQ-0008 -->

When a Flight's departureTime passes (becomes less than or equal to the
simulator's current clock), the simulator SHALL remove that Flight from
the in-memory store. As part of the same eviction operation, the
simulator MUST also remove every PassengerRecord whose parent flight is
the evicted Flight, and every BaggageRecord whose parent passenger is
one of those evicted PassengerRecords. The eviction MUST be atomic from
the perspective of any concurrent reader: at no observable point may a
PassengerRecord or BaggageRecord remain in the store after its parent
Flight has been removed. This requirement encodes FND-0001's "evict on
departure" decision and the parent–child cascade implied by the
ENT-0001 → ENT-0002 → ENT-0003 ownership chain.

#### Scenario: A flight is removed from the store within one simulator tick of departure

- GIVEN a Flight F in the in-memory store whose departureTime is exactly
  the simulator's next clock tick
- WHEN the simulator advances its clock past F's departureTime
- THEN within the same simulator-tick boundary, the in-memory store no
  longer contains F (a query for the Flight by flightId returns no row)

#### Scenario: A flight's eviction cascades to its passenger records

- GIVEN a Flight F has been evicted at its departureTime, and prior to
  eviction F had 250 PassengerRecord rows associated with it
- WHEN the in-memory store is queried for PassengerRecord rows whose
  parent flight identifier equals F's flightId
- THEN the query returns zero rows

#### Scenario: A flight's eviction cascades to its baggage records

- GIVEN a Flight F has been evicted at its departureTime, and prior to
  eviction F had 250 PassengerRecords with 400 BaggageRecord rows total
  spread across them
- WHEN the in-memory store is queried for BaggageRecord rows whose
  parent passenger identifier is one of F's former PassengerRecord IDs
- THEN the query returns zero rows
