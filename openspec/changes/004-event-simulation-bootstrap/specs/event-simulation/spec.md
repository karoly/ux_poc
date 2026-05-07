# event-simulation: Continuous passenger and baggage events

## Purpose

This capability defines the continuous progression of passengers
through check-in and of baggage through its registration / drop-off
states on top of the active flight pool maintained by capability
`flight-generation`. It owns three update-axis behaviours: assigning a
PassengerRecord to a check-in channel (online or counter) and stamping
its check-in time; assigning a seat to a checked-in passenger; and
moving a BaggageRecord through its `registered` then `dropped` states.
Each event rolls the matching Flight aggregate forward and emits a
state-change notification to the Event Stream Server (per the Ch 4
flow ENT-0001: MOD-0001 → MOD-0002, async-at-most-once). Bootstrapping
and eviction of these entities is governed by capability
`flight-generation` and is out of scope here.

## Dependencies

- Foundation decisions: FND-0002, FND-0005
- Modules: MOD-0001, MOD-0002
- Entities: ENT-0001, ENT-0002, ENT-0003

## Requirements

### Requirement: Passenger check-in events progress a PassengerRecord and update its Flight's channel aggregate

<!-- forge-id: REQ-0013 -->

When the simulator generates a passenger check-in event for a
PassengerRecord P that is currently un-checked-in (P.checkInChannel
and P.checkInTime are both null), the simulator SHALL set
P.checkInChannel to either `online` or `counter`, set P.checkInTime to
the simulator's current clock value, and increment the parent Flight's
checkedInOnline aggregate (if the chosen channel is online) or
checkedInCounter aggregate (if counter) by exactly 1. The simulator
SHALL NOT generate a check-in event for a PassengerRecord whose
checkInChannel is already non-null (each passenger checks in at most
once). The choice of channel (online vs. counter) for a given
passenger MAY be drawn from any distribution the simulator's
parameters permit; the L2 contract pins only the post-condition
relationship between the event and the recorded state.

#### Scenario: Online check-in increments Flight.checkedInOnline by exactly 1

- GIVEN a Flight F with checkedInOnline = 50 and a PassengerRecord P
  on F where P.checkInChannel is null
- WHEN the simulator generates a check-in event for P with channel
  `online`
- THEN P.checkInChannel = `online`, P.checkInTime is non-null and
  equals the simulator's current clock value at event time, and F's
  checkedInOnline equals 51 (no other Flight aggregate is changed by
  this event)

#### Scenario: Counter check-in increments Flight.checkedInCounter by exactly 1

- GIVEN a Flight F with checkedInCounter = 30 and a PassengerRecord P
  on F where P.checkInChannel is null
- WHEN the simulator generates a check-in event for P with channel
  `counter`
- THEN P.checkInChannel = `counter`, P.checkInTime is non-null, and
  F's checkedInCounter equals 31 (and checkedInOnline is unchanged)

#### Scenario: A PassengerRecord cannot be checked in twice

- GIVEN a PassengerRecord P with checkInChannel already set to
  `online`
- WHEN the simulator iterates through eligible passengers for the
  next check-in event
- THEN P is not selected (no second check-in event is generated for P;
  P.checkInChannel and P.checkInTime remain unchanged across the
  remainder of the run)

### Requirement: Seat-assignment events update PassengerRecord.seatAssigned and Flight.seatsAssigned

<!-- forge-id: REQ-0014 -->

When the simulator generates a seat-assignment event for a
PassengerRecord P, the simulator SHALL set P.seatAssigned to a
non-null seat string drawn from the parent Flight's seat inventory
(the seat string SHALL NOT collide with any other PassengerRecord's
seatAssigned on the same Flight) and SHALL increment the parent
Flight's seatsAssigned aggregate by exactly 1. A PassengerRecord's
seatAssigned MUST NOT be reassigned once non-null; the simulator
SHALL NOT generate a seat-assignment event for a passenger whose
seatAssigned is already non-null. The relationship between check-in
and seat assignment (whether seat assignment must follow check-in,
precede it, or runs independently) is a simulator-parameter
concern; this requirement governs only the per-event post-condition.

#### Scenario: Seat assignment increments Flight.seatsAssigned and is non-colliding

- GIVEN a Flight F with seatsAssigned = 80 and totalSeats = 250, with
  PassengerRecord P on F where P.seatAssigned is null and where 80
  other passengers on F already have non-null seatAssigned values
- WHEN the simulator generates a seat-assignment event for P
- THEN P.seatAssigned is non-null, no other PassengerRecord on F has
  the same seatAssigned value as P, F's seatsAssigned equals 81, and
  no other Flight aggregate is changed

#### Scenario: A PassengerRecord cannot be re-seated

- GIVEN a PassengerRecord P with seatAssigned already set to "12A"
- WHEN the simulator iterates through eligible passengers for the
  next seat-assignment event
- THEN P is not selected (P.seatAssigned remains "12A" and the parent
  Flight's seatsAssigned does not increment from another event aimed
  at P)

### Requirement: Baggage events progress BaggageRecord state through registered then dropped

<!-- forge-id: REQ-0015 -->

When the simulator generates a baggage-registration event for a
BaggageRecord B whose status is the initial pre-registered state, the
simulator SHALL transition B.status to `registered` and SHALL
increment B's parent Flight's bagsRegistered aggregate by exactly 1.
When the simulator generates a baggage-drop event for a BaggageRecord
B whose status is `registered`, the simulator SHALL transition
B.status to `dropped` and SHALL increment that Flight's bagsDropped
aggregate by exactly 1. The simulator MUST NOT skip the registered
state — a BaggageRecord transitions in the order
pre-registered → registered → dropped — and MUST NOT regress (a
BaggageRecord that has reached `dropped` does not return to any
earlier state). Per Ch 2 the bagsDropped count is always less than or
equal to the bagsRegistered count.

#### Scenario: Registration progresses status and increments bagsRegistered

- GIVEN a Flight F with bagsRegistered = 100, bagsDropped = 80, and a
  BaggageRecord B on F whose status is the initial pre-registered
  state
- WHEN the simulator generates a baggage-registration event for B
- THEN B.status = `registered`, F's bagsRegistered = 101, F's
  bagsDropped is unchanged at 80

#### Scenario: Drop progresses status from registered and increments bagsDropped

- GIVEN a Flight F with bagsRegistered = 101, bagsDropped = 80, and a
  BaggageRecord B on F whose status is `registered`
- WHEN the simulator generates a baggage-drop event for B
- THEN B.status = `dropped`, F's bagsDropped = 81, F's bagsRegistered
  is unchanged at 101

#### Scenario: A BaggageRecord cannot skip registered or regress from dropped

- GIVEN a BaggageRecord B in initial pre-registered state and a
  BaggageRecord C with status = `dropped`
- WHEN the simulator iterates through eligible baggage records for the
  next event
- THEN B may be eligible only for a registration event (not a drop
  event) and C is not eligible for any further event (no
  drop-then-register or dropped-back-to-registered transition is
  generated)

### Requirement: Each state-changing event emits a Flight notification to the Event Stream Server

<!-- forge-id: REQ-0016 -->

For every state change defined by REQ-0013, REQ-0014, or REQ-0015, the
simulator SHALL emit exactly one notification to the Event Stream
Server (capability `state-broadcast` consumes it). The notification
payload MUST carry the parent Flight's flightId, checkedInOnline,
checkedInCounter, bagsRegistered, bagsDropped, and seatsAssigned —
verbatim with the field set named in the Ch 4 flow ENT-0001:
MOD-0001 → MOD-0002, async-at-most-once. Notification delivery is
async-at-most-once: the simulator SHALL NOT block waiting for
acknowledgement, MAY drop a notification on transient back-pressure,
and SHALL NOT retry. This requirement encodes the consistency posture
declared on the Ch 4 flow.

#### Scenario: A check-in event produces exactly one notification with the flow-defined fields

- GIVEN a PassengerRecord P on a Flight F with the relevant pre-event
  aggregates
- WHEN the simulator generates a check-in event for P that updates F's
  checkedInOnline aggregate
- THEN the Event Stream Server receives exactly one notification whose
  payload contains F's flightId together with F's post-event
  checkedInOnline, checkedInCounter, bagsRegistered, bagsDropped, and
  seatsAssigned values — and contains no other Flight's data

#### Scenario: Notification delivery does not block the simulator's next event

- GIVEN the Event Stream Server is artificially blocked by a test
  harness so it cannot drain notifications
- WHEN the simulator generates a sequence of N state-changing events
- THEN every event's in-memory state mutation completes within its
  expected time window (the simulator does not stall waiting for the
  Event Stream Server) and notifications that cannot be delivered may
  be dropped (no retry queue grows unboundedly)

### Requirement: Event generation rate is steady and uniform with no peak-load bursts

<!-- forge-id: REQ-0017 -->

The simulator SHALL generate state-changing events at a steady,
uniform rate over the lifetime of the process. The aggregate event
arrival rate (passenger check-ins, seat assignments, baggage
registrations, baggage drops, summed across the active flight pool)
MUST NOT exhibit a burst pattern — defined here as a window during
which the arrival rate exceeds the simulator's stated steady ceiling
by more than the bounded variance the simulator's parameters permit.
This requirement encodes the Ch 2 hard constraint "steady, uniform
arrival of events (no peak-load bursts)".

#### Scenario: Inter-arrival intervals fall within the bounded variance band

- GIVEN the simulator has been running for at least one full simulator
  hour
- WHEN the test harness records the inter-arrival interval between
  every consecutive pair of state-changing events over the most recent
  one-hour window
- THEN every interval falls within the bounded variance band that the
  simulator's parameters specify around the mean inter-arrival time
  (no observed interval undershoots the lower bound; no peak burst is
  observed)

#### Scenario: The aggregate per-second event rate stays at or below the steady ceiling

- GIVEN the simulator has been running for at least one full simulator
  hour
- WHEN the test harness counts state-changing events per wall-clock
  second across the most recent one-hour window
- THEN no one-second slot's count exceeds the simulator's stated
  steady ceiling (events per second) by more than the bounded
  variance the simulator's parameters permit
