# state-broadcast: Push flight status events to all connected browsers

## Purpose

This capability defines the producer side of the server-to-browser
push path: the Event Stream Server receives a state-change
notification from the Flight Data Simulator (per the Ch 4 flow
ENT-0001: MOD-0001 → MOD-0002), serialises it as a text/event-stream
event, and pushes that event across the connection registry that
capability `sse-connection-management` maintains (per the Ch 4 flow
ENT-0001: MOD-0002 → MOD-0003 via IFC-0001 / IFC-0002). The
end-to-end latency from the simulator's state mutation to the
browser's receipt of the event must respect the 5-second ceiling that
Ch 2 § Constraint pins as a hard functional requirement of the PoC.
Connection acceptance and lifecycle handling are governed by
capability `sse-connection-management` and are out of scope here;
notification production is governed by capability `event-simulation`
and is out of scope here.

## Dependencies

- Foundation decisions: FND-0002, FND-0003, FND-0004
- Modules: MOD-0001, MOD-0002, MOD-0003
- Interfaces: IFC-0001, IFC-0002

## Requirements

### Requirement: Each received notification is broadcast across every currently-connected client

<!-- forge-id: REQ-0022 -->

When the Event Stream Server receives a state-change notification from
the simulator (per the consumer-side Ch 4 flow ENT-0001:
MOD-0001 → MOD-0002), the broadcast SHALL iterate the current
connection registry maintained by capability
`sse-connection-management` and SHALL emit one
text/event-stream event derived from the notification on each
currently-open connection at the moment of broadcast. Connections
that are added to the registry after broadcast begins for a given
notification SHALL NOT receive that particular notification (no
catch-up / no replay; the no-backlog posture from CAP-0003 holds at
the broadcast layer too). Connections that close during broadcast
SHALL NOT prevent the broadcast from completing for the remaining
connected clients — a per-connection failure SHALL be isolated.

#### Scenario: A single notification reaches every connected client

- GIVEN three test-harness clients are connected and the registry's
  count of currently-connected clients equals 3
- WHEN the simulator emits a single state-change notification for a
  Flight F
- THEN each of the three test-harness clients receives exactly one
  event for F over their respective connections (no client receives
  zero, no client receives more than one for this notification)

#### Scenario: A failed write to one client does not stop delivery to peers

- GIVEN three test-harness clients are connected, and one of the three
  has had its TCP connection forcibly broken in a way the server has
  not yet detected
- WHEN the simulator emits a single state-change notification for a
  Flight F
- THEN the broadcast loop completes; the two healthy clients each
  receive exactly one event for F; the failure observed against the
  broken connection does not prevent delivery to the two healthy
  peers

### Requirement: Each event payload carries the six flow-defined fields

<!-- forge-id: REQ-0023 -->

For every text/event-stream event emitted by the broadcast, the
event's data field MUST carry the six fields named on the Ch 4 flows
ENT-0001: MOD-0001 → MOD-0002 and ENT-0001: MOD-0002 → MOD-0003 —
flightId, checkedInOnline, checkedInCounter, bagsRegistered,
bagsDropped, and seatsAssigned — sourced verbatim from the received
notification. The serialisation format MUST be one a browser
EventSource can parse from a single SSE event boundary (a JSON object
encoded as a single SSE `data:` field is the conventional shape;
this requirement does not pin the encoding beyond "browser-EventSource
parsable as a single event" so the L3 build retains room to choose).
No additional flight identifier or aggregate is REQUIRED in the
payload, and the broadcast MUST NOT redact any of the six fields
the notification supplies.

#### Scenario: An emitted event contains exactly the six flow-defined fields

- GIVEN the simulator emits a notification for a Flight F whose six
  flow-defined fields are populated (flightId "AF1234",
  checkedInOnline 121, checkedInCounter 40, bagsRegistered 130,
  bagsDropped 110, seatsAssigned 161)
- WHEN a connected test-harness client reads the resulting event off
  the SSE stream
- THEN the parsed event yields a mapping containing those six fields
  with those exact values

#### Scenario: A test-harness EventSource can parse the emitted event boundary

- GIVEN a connected test-harness client built on the browser
  EventSource API
- WHEN the simulator emits a state-change notification and the server
  broadcasts the corresponding event
- THEN the EventSource fires a single message event whose `data`
  property is parsable as the six-field mapping (no event boundary
  ambiguity, no truncation, no multi-event split for a single
  notification)

### Requirement: End-to-end broadcast latency stays at or below the 5-second ceiling at p95

<!-- forge-id: REQ-0024 -->

For at least 95% of state-change notifications observed across any
sustained sampling window of at least one simulator hour, the elapsed
time between the simulator's state mutation completing and a
connected client receiving the corresponding event over the SSE
stream MUST be at or below 5000 milliseconds. This requirement
encodes the Ch 2 hard constraint "Display latency ceiling of 5
seconds" and the IFC-0001 / IFC-0002 SLO contract "p95 update
delivery < 5000ms".

#### Scenario: p95 latency is at or below 5 seconds across a one-hour window

- GIVEN the simulator and the Event Stream Server are running steady
  state with at least one connected test-harness client subscribing to
  the SSE stream, and the test harness records, for each notification
  the simulator emits, the wall-clock interval between the simulator's
  mutation timestamp and the client's event-receipt timestamp
- WHEN the harness completes a continuous one-hour sampling window
- THEN the 95th percentile of the recorded intervals is at or below
  5000 milliseconds

#### Scenario: A single notification under unloaded conditions is well within budget

- GIVEN the simulator and the Event Stream Server are running with
  exactly one connected client and an idle event rate
- WHEN the simulator emits a single state-change notification
- THEN the connected client receives the corresponding event within
  the system's nominal sub-second latency window (and well within
  the 5000ms ceiling — a single-notification-under-no-load case
  represents the floor, not the ceiling)

### Requirement: Broadcast is async-at-most-once with no retry and may drop on back-pressure

<!-- forge-id: REQ-0025 -->

The broadcast SHALL implement the async-at-most-once consistency
posture declared on the Ch 4 flow ENT-0001: MOD-0002 → MOD-0003.
Specifically: the broadcast SHALL NOT retry a per-client send that
fails, SHALL NOT buffer undelivered events for later replay, and MAY
drop a notification rather than block the simulator if the broadcast
or any per-client send is back-pressured. On a dropped notification
or a per-client send failure, the broadcast SHALL emit a structured
log line to standard output (per FND-0004) so the failure is visible
in operator logs, but SHALL NOT emit any error payload over the SSE
stream to peers (per FND-0002 — passive degradation; peer clients
see no event for that drop and no error).

#### Scenario: A per-client send failure is not retried for that client

- GIVEN a test-harness client whose SSE socket has been broken in a
  way that the next per-client write will fail
- WHEN the simulator emits a state-change notification and the
  broadcast attempts to deliver an event to that client and fails
- THEN the broadcast does not enqueue, schedule, or otherwise retry
  the failed send; the failure is logged to stdout in structured
  form, the client is removed from the registry per CAP-0003's
  disconnect handling, and no replay buffer accumulates events
  intended for that client

#### Scenario: A drop under back-pressure does not block the simulator's next event

- GIVEN the broadcast has been artificially back-pressured by a test
  harness so the server cannot drain notifications at the rate the
  simulator produces them
- WHEN the simulator emits a sequence of N state-change notifications
- THEN the simulator's per-event mutation completes within its
  expected time window for each of the N events (no event blocks
  waiting on the broadcast); some notifications MAY be observably
  missing from connected clients' streams (drops); each drop is
  recorded as a structured log line on stdout and no error payload is
  emitted to connected clients
