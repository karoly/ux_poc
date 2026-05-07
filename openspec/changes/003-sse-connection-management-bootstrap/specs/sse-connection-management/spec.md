# sse-connection-management: Browser connection lifecycle

## Purpose

This capability defines the connection-lifecycle contract of the Event
Stream Server: how new browser clients are accepted onto the
server-push channel (per FND-0003), how the server tracks the set of
currently-connected clients so that capability `state-broadcast` has
something to iterate over, how disconnects are handled (per FND-0002's
passive-degradation posture), and the explicit posture that no event
replay or backlog is offered to newly-arriving clients. The actual
broadcasting of flight state changes across the connected set is
governed by capability `state-broadcast` and is out of scope here.

## Dependencies

- Foundation decisions: FND-0002, FND-0003, FND-0004
- Modules: MOD-0002
- Interfaces: IFC-0001, IFC-0002
- Actors: ACT-0001, ACT-0002

## Requirements

### Requirement: Endpoint accepts persistent server-push client connections

<!-- forge-id: REQ-0009 -->

The server SHALL expose an HTTP endpoint that accepts client
connections and responds with a persistent server-push response stream.
On accepting a new connection, the server SHALL respond with HTTP
status 200 and the response headers MUST set Content-Type to
`text/event-stream`, set Cache-Control to a value that disables
intermediate caching of the stream (such as `no-cache`), and SHALL
keep the underlying connection open for the duration of the client
session rather than closing after the response headers are sent. This
requirement encodes the FND-0003 § Verified-by check (the SSE endpoint
returns Content-Type: text/event-stream).

#### Scenario: Endpoint responds with the SSE response shape

- GIVEN the Event Stream Server is running and a test harness opens a
  new HTTP connection to the SSE endpoint
- WHEN the response headers arrive
- THEN the response status is 200, the Content-Type header equals
  `text/event-stream`, and the Cache-Control header is set to a value
  that disables intermediate caching

#### Scenario: Connection is held open after headers are sent

- GIVEN a test harness has just received the SSE response headers
- WHEN the harness reads from the response body for at least 30
  consecutive seconds without sending any further request bytes
- THEN the underlying TCP connection remains open throughout (read
  operations do not return end-of-stream, and the server does not
  close the connection)

### Requirement: Server tracks the set of currently-connected clients

<!-- forge-id: REQ-0010 -->

The server MUST maintain an in-memory registry of every currently-open
SSE connection. When a new client connects (per REQ-0009), the
connection SHALL be added to the registry before any subsequent
broadcast operation runs (capability `state-broadcast` reads from this
registry to fan out events; if the new connection is not present in
the registry, the client cannot receive events). The registry SHALL be
queryable for the count of currently-connected clients so capability
`state-broadcast` and the structured-logging surface (FND-0004) can
report the connected-client count at lifecycle events.

#### Scenario: Newly-connected client appears in the registry

- GIVEN the server starts with the registry empty (zero connected
  clients)
- WHEN a single test-harness client opens a new connection to the SSE
  endpoint and reads the response headers
- THEN the registry's count of currently-connected clients equals 1

#### Scenario: Multiple clients are tracked independently

- GIVEN the server starts with the registry empty
- WHEN three test-harness clients open three separate connections
  concurrently and each reads the response headers
- THEN the registry's count equals 3, and each of the three connections
  is independently identifiable in the registry such that a per-client
  send operation can target one without affecting the others

### Requirement: Client disconnect closes the connection cleanly and removes the registry entry

<!-- forge-id: REQ-0011 -->

When a client connection is terminated (the client closes its side of
the TCP connection, the network link drops, or the server detects a
write failure to the client), the server SHALL close its side of the
connection cleanly within a bounded server-side reaction window and
SHALL remove the corresponding registry entry. After removal, the
registry's count of currently-connected clients reflects the new total
and capability `state-broadcast` SHALL NOT attempt further sends to
the removed connection. No error payload SHALL be emitted to the
remaining connected clients on a peer disconnect (the FND-0002
passive-degradation posture means a disconnect is a non-event from
peers' perspective).

#### Scenario: Registry shrinks when a client closes its connection

- GIVEN three test-harness clients are connected and the registry
  count equals 3
- WHEN one of the three clients closes its TCP connection from the
  client side
- THEN the registry count returns to 2 (the server detects the close
  and removes the entry), and the remaining two connections continue
  to be readable without disruption

#### Scenario: No error event is emitted to peers on a disconnect

- GIVEN three test-harness clients are connected and reading the
  response body
- WHEN one of the three closes its connection
- THEN neither of the remaining two clients receives any event payload
  describing the third client's disconnect (no peer-disconnect event
  is part of the protocol surface)

### Requirement: No event replay or backlog is offered to newly-arriving clients

<!-- forge-id: REQ-0012 -->

The server SHALL NOT maintain any buffer of past events for replay to
newly-arriving or reconnecting clients. A client that connects at time
T SHALL receive only events that capability `state-broadcast`
publishes at or after T over the lifetime of that connection;
specifically, the server SHALL NOT respond to the SSE
`Last-Event-ID` request header by replaying historical events, and
the server SHALL NOT issue or honour any explicit `id:` field on
emitted events that would allow a client to request a resume point.
This requirement encodes the explicit MOD-0002 description ("no event
replay is provided on reconnection") and the passive-degradation
posture from FND-0002 — clients that miss events because of a
disconnect see only the next live update once they reconnect.

#### Scenario: A reconnecting client does not receive missed events

- GIVEN a client A connected at time T0, and capability
  `state-broadcast` published five events between T0 and T1
- WHEN client A's connection drops at T1, A reconnects at T2 (with T2
  > T1), and a sixth event is published at T3 (with T3 > T2)
- THEN over A's new connection, A receives only the event published at
  T3 (and any later events); A does not receive the five events
  published between T0 and T1

#### Scenario: A Last-Event-ID request header does not trigger a replay

- GIVEN capability `state-broadcast` has published several events over
  some prior connection
- WHEN a new test-harness client opens a connection with the
  `Last-Event-ID` request header set to a value the server might
  recognise
- THEN the server's response stream contains no events that pre-date
  the new connection's open time; the header is effectively ignored
  by the server's reply logic
