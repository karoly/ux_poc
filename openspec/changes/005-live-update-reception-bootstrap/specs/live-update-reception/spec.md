# live-update-reception: Receive and apply incoming status events

## Purpose

This capability defines how the dashboard establishes and maintains
its persistent connection to the Event Stream Server, parses the
server-pushed flight-status events, and applies each event to the
client-side state that capability `dashboard-layout` renders. The
end-to-end behaviour the user observes — the grid reflects new
check-in counts without any page reload — depends on three contracts
captured here: the connection-establishment posture, the parse shape
of each event, and the application step that mutates client state by
flightId. The static layout structure of the grid itself is governed
by capability `dashboard-layout` and is out of scope here.

## Dependencies

- Foundation decisions: FND-0002, FND-0003, FND-0006
- Modules: MOD-0003
- Interfaces: IFC-0001, IFC-0002
- Actors: ACT-0001, ACT-0002

## Requirements

### Requirement: Open a persistent SSE connection to the Event Stream Server on dashboard mount

<!-- forge-id: REQ-0018 -->

When the dashboard application mounts in the browser, the application
SHALL open a persistent server-push connection to the Event Stream
Server's SSE endpoint (the endpoint described in capability
`sse-connection-management`). The connection SHALL remain open for the
lifetime of the dashboard's mounted state and SHALL be re-opened
automatically by the browser's native server-push retry behaviour if
the underlying TCP connection is lost (per FND-0003 — "Native browser
support with automatic retry on disconnect"). The dashboard SHALL NOT
issue any client-to-server message body on this connection — the
stream is unidirectional, server-to-client only.

#### Scenario: Connection opens within the mount sequence

- GIVEN the dashboard application is loaded in a browser and has not
  previously opened a connection
- WHEN the dashboard's mount sequence completes
- THEN exactly one SSE connection is observed open from the dashboard
  to the server-push endpoint, and the connection's response headers
  include Content-Type `text/event-stream`

#### Scenario: No client-to-server message body is sent on the SSE connection

- GIVEN the dashboard has opened its SSE connection
- WHEN the test harness inspects the bytes the client transmits over
  the connection over a 30-second window
- THEN the client transmits only the original request line and headers
  (no client-side message body, no follow-up writes, no application-
  level keepalive payload)

### Requirement: Parse each incoming event into the flow-defined field set

<!-- forge-id: REQ-0019 -->

For every event the SSE connection delivers, the dashboard SHALL parse
the event payload and extract the field set named in the Ch 4 flow
ENT-0001: MOD-0002 → MOD-0003 — flightId, checkedInOnline,
checkedInCounter, bagsRegistered, bagsDropped, and seatsAssigned. An
event whose payload is missing any of those six fields MUST be
discarded (no partial state application is permitted; the client-side
state retains the previous values for that flight). An event whose
payload contains additional fields MAY be applied; the additional
fields are ignored. Discarded events SHALL NOT trigger an error
indicator in the UI (consistent with FND-0002's passive-degradation
posture).

#### Scenario: A complete event is parsed into the six named fields

- GIVEN the dashboard has an open SSE connection
- WHEN the server pushes an event whose payload contains all six
  flow-defined fields with valid values (e.g., flightId "AF1234",
  checkedInOnline 121, checkedInCounter 40, bagsRegistered 130,
  bagsDropped 110, seatsAssigned 161)
- THEN the parser produces a parsed event with exactly those six
  values bound to their named fields, ready for the application step

#### Scenario: An incomplete event is discarded without UI side effect

- GIVEN the dashboard has an open SSE connection and the client-side
  state for flight AF1234 has checkedInOnline = 120
- WHEN the server pushes an event whose payload omits the
  checkedInCounter field
- THEN no field of the client-side state for AF1234 is mutated by this
  event (checkedInOnline remains 120 until a complete subsequent event
  arrives), and no error or warning indicator is rendered in the UI

### Requirement: Apply each parsed event to client-side state, keyed by flightId

<!-- forge-id: REQ-0020 -->

For every successfully-parsed event (per REQ-0019), the dashboard
SHALL update the client-side state entry whose flightId matches the
event's flightId by replacing that entry's checkedInOnline,
checkedInCounter, bagsRegistered, bagsDropped, and seatsAssigned values
with the corresponding values from the event. If no client-side state
entry exists for the event's flightId at the moment the event is
processed, the dashboard SHALL create one (so that flights introduced
into the simulator's pool after the dashboard mounted appear on the
display once their first event arrives). The state mutation SHALL
trigger a re-render of the affected flight card such that the rendered
DOM reflects the new values within the bounded UI-render window — and
SHALL NOT trigger a full page reload.

#### Scenario: An event for an existing flight updates that flight's state in place

- GIVEN the client-side state contains an entry for flightId "AF1234"
  with checkedInOnline = 120
- WHEN a parsed event arrives carrying flightId "AF1234" and
  checkedInOnline = 121 (with the other four fields also set)
- THEN the client-side state entry for AF1234 has checkedInOnline = 121
  (and the other four fields equal to the event's values), no other
  flight's state entry is mutated, and no full page reload occurs

#### Scenario: An event for an unknown flight inserts a new state entry

- GIVEN the client-side state contains no entry for flightId "BA9999"
- WHEN a parsed event arrives carrying flightId "BA9999" and the five
  status fields populated
- THEN a new client-side state entry for BA9999 is created carrying
  those five fields, and the rendered grid contains a new flight card
  for BA9999 within the bounded UI-render window

### Requirement: On stream interruption, freeze the displayed state without an error indicator

<!-- forge-id: REQ-0021 -->

If the SSE connection is interrupted (the underlying TCP connection
closes from either side, the network drops, or the server stops
pushing events for any reason), the dashboard SHALL retain the
client-side state at its last-received values and SHALL NOT render any
error, warning, or stale-data indicator in the UI. The browser's
native server-push retry behaviour MAY re-establish the connection;
once re-established, REQ-0019 / REQ-0020 resume applying events
without any backfill of events missed during the interruption (per the
no-replay posture of capability `sse-connection-management`). This
requirement encodes FND-0002's passive-degradation posture for
MOD-0003.

#### Scenario: A dropped connection leaves the rendered grid frozen at last-received values

- GIVEN the client-side state contains entries for 100 flights and
  every flight card on screen reflects its last-received values
- WHEN the SSE connection is forcibly closed by a test harness and no
  new events arrive for 30 seconds
- THEN every rendered flight card continues to display the values it
  had immediately before the disconnect, and no error or warning
  indicator appears anywhere in the rendered DOM

#### Scenario: Reconnection resumes event application without backfilling missed events

- GIVEN the SSE connection has been forcibly closed and five events
  were dispatched by the server during the disconnect window
- WHEN the browser's native retry re-establishes the connection and a
  fresh event for flightId "AF1234" then arrives
- THEN the client-side state for AF1234 reflects the values from the
  fresh event (the five missed events are not delivered or applied),
  and other flights' state entries are unchanged from their pre-
  disconnect values until their next live event arrives
